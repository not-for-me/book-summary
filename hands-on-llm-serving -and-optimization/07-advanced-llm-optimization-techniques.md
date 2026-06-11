# 07. Advanced LLM Optimization Techniques

## 챕터 개요 (3줄 요약)
- speculative decoding은 작은 draft 모델/모듈이 token을 추측하고 target 모델이 병렬 검증해 decode phase의 ITL을 2~3배 단축한다(정확도 무손실).
- 대형 LLM은 multi-GPU/multi-node 병렬화(DP·TP·PP·EP)와 prefill-decode disaggregation으로 서빙하며, 각 단계를 독립 최적화·scaling할 수 있다.
- advanced KV caching(long-context CAG·offloading·compression·blending)으로 TTFT를 대폭 줄이고, 현대 serving은 kernel→engine→cache→routing의 다층 stack으로 진화했다.

---

## 1. Speculative Decoding — 개념과 단계
> draft 모델이 K개 token을 빠르게 추측하고 target 모델이 forward pass로 병렬 검증해, 수용 시 건너뛰어 decode를 가속한다(최종 출력은 비-speculative와 동일).
- 검증: draft 확률 0.6, target 0.8이면 수용; target이 더 낮으면 확률적(0.4/0.6) 수용. 거부 시 이후 draft 폐기(autoregressive)하고 target이 보정 분포에서 새 token 샘플링.
- K는 핵심 파라미터 — 정확도는 verification으로 보장.

---

## 2. Speculative Decoding — Tuning & 방법
> 속도와 acceptance rate를 최대화하는 작은 모델 선택이 핵심이다.
- draft 모델: 같은 family(동일 tokenizer·pretraining)로 acceptance↑, 공격적 quantization 가능(target이 fallback). target에서 distill하면 더 우수.
- self-drafting: 외부 draft 없이 모델 자체가 추측 — Medusa(다중 prediction head로 후속 token 병렬 생성), EAGLE(hidden state 예측, EAGLE-2 dynamic draft tree, EAGLE-3 다층 feature fusion; 최고 성능이나 추가 학습 필요).
- n-gram: 요청 앞부분에서 인접 n token을 table로 저장·매칭(모델 없음·저오버헤드). JSON/SQL·템플릿 채우기 등 결정적·반복 패턴에 강력 → 첫 시도 권장.
- K 튜닝: 크면 상한↑이나 acceptance 낮으면 낭비, 작으면 안정적. 보통 4~8, 구조화 출력은 16~32. position별 acceptance([0.8,0.7,...,0.02]) 관찰해 조정.

---

## 3. Speculative Decoding — 한계와 벤치
> decode stage만 도움이 되며, acceptance가 낮거나 prefill이 길어 이미 compute-bound면 효과가 없을 수 있다.
- 대형 batch는 memory-bound→compute-bound로 전환되어 latency는 개선되나 throughput은 추가 연산으로 악화.
- 두 모델을 공유 GPU에서 효율 운영이 어려워 최근 self-drafting·n-gram에 집중. 정적 K는 변화 포착 한계.
- 벤치(Qwen3-32B): concurrency=1에서 improved n-gram ~16%, EAGLE-3 ~2배(56.5 vs 28.9 tok/s). concurrency=16에선 n-gram이 vanilla보다 나빠지고 EAGLE-3 이득 축소·TTFT 증가(speculative head 오버헤드). 적합: latency 민감 + throughput/TTFT 일부 희생 가능.

---

## 4. Multi-GPU/Multi-Node — Data Parallelism
> 단일 GPU 메모리/성능을 초과하는 대형 LLM 서빙은 분산 전략(DP·TP·PP·EP)이 필요하다.
- DP: 모델 instance를 복제해 트래픽을 분산(throughput·고가용성). 인스턴스 down 시 router가 나머지로 라우팅.
- routing: round robin, least connections, latency-based, cache-aware(prefix가 저장된 인스턴스로). LLM routing은 KV cache locality·hit율·공간 사용률·queue·생성 token 수·SLA budget 등 실시간 신호 활용(NVIDIA Dynamo KV Router·llm-d).

---

## 5. Tensor Parallelism & Pipeline Parallelism
> TP는 모델을 width로(각 layer 가로 분할), PP는 depth로(layer 블록 세로 분할) 샤딩한다.
- TP: 각 layer를 GPU에 분할, 부분 계산 후 결합 — layer마다 inter-GPU 통신 필요. PP: layer 그룹을 GPU에 배치, assembly line처럼 순차 전달 — 통신 적으나 pipeline bubble(느린 stage 대기) 발생.
- 의사결정: 단일 GPU에 들어가면 통신 오버헤드 회피 위해 유지(GPU 추가는 비선형 scaling), quantization 먼저 고려. 단일 node 내 NVLink 있으면 TP 이상적, 없으면(PCIe 느림) PP/작은 TP.
- hyperscale: node 내 TP(NVLink) + node 간 PP(느린 inter-node 통신 최소화). EP·PD disaggregation도 유효. vLLM: --tensor-parallel-size, --pipeline-parallel-size; multi-node는 Ray + vLLM.

---

## 6. Expert Parallelism
> MoE 모델(Mixtral·DeepSeek-V3·GPT-OSS)은 token을 동적으로 expert에 라우팅해 latency·cost를 줄이나 총 파라미터가 단일 GPU에 안 들어갈 수 있다.
- EP: expert를 여러 GPU에 균등 분산, token은 선택된 expert가 적재된 GPU로만 dispatch(비활성 expert 연산 회피). TP·PP를 보완해 함께 최적 성능.

---

## 7. Prefill-Decode Disaggregation — 이점
> prefill(compute-bound)과 decode(memory bandwidth-bound)를 별도 GPU로 분리해 서로 다른 자원 프로파일의 간섭을 제거한다.
- TTFT·ITL 독립 최적화: 비대칭 자원 할당(input-heavy면 prefill 자원↑로 TTFT↓), 간섭 없어 ITL 안정.
- workload 독립 최적화: idle 제거, prefill/decode별 batch size·TP/PP 전략 차등 적용.
- 하드웨어 유연성: prefill은 compute-optimized(H100), decode는 memory-optimized(H200) 또는 저가(L40S)로 비용 절감.
- 독립·비대칭 scaling: prefill은 burst·공격적, decode는 예측 가능·안정.

---

## 8. PD Disaggregation — 아키텍처 & KV Cache Transfer
> DistServe 구조: controller가 prefill instance로 라우팅 → KV cache 생성 → decode instance로 전송 → token 생성. KV cache transfer 효율이 성패를 좌우한다.
- KV cache는 매우 큼(8B 모델·1,024 token ≈ 0.1~0.15GB; 입력 10배·16 req/s면 ~25GB/s 전송 필요). NVLink(900GB/s) > InfiniBand RDMA(50~100GB/s) > PCIe(~10GB/s).
- 동일 node 내 NVLink 배치가 이상적이나 TP 등으로 inter-node 불가피 시 RDMA 필수 + 최적화: chunk 전송(streaming), async non-blocking(compute-통신 중첩), layer 단위 전송, KV cache 압축.
- 최적화 시 PD 오버헤드를 per-request latency의 1% 미만으로 감소. 사용 시점: 대형 모델·고부하·고급 TTFT/ITL 튜닝 필요 시만, 소형은 단순 setup.

---

## 9. Advanced KV Caching — Long-Context Serving
> coding copilot·conversational agent·enterprise 등 방대한 context 추론 수요가 long-context 서빙을 요구한다.
- RAG: query 시 관련 문서 retrieve해 prompt에 주입 → context·TTFT 관리, 최신·tenant 지식 grounding.
- CAG: 관련 context 대부분/전부를 KV cache로 캐싱해 요청 간 재사용. prefix caching은 naive CAG(static prefix + dynamic suffix).
- 가능해진 배경: long-context 모델(100k~1M token, "lost in the middle" 완화) + 효율적 KV cache 관리(CPU/SSD/원격 offloading, cross-replica routing). retrieval 없이 redundant 연산 감소 → RAG보다 빠른 TTFT.

---

## 10. Cost/Latency 계산 (RAG vs CAG)
> RAG는 동적 chunk로 cache hit율 낮아 non-cached prefill이 많고, CAG는 static 긴 context를 캐싱해 user prompt만 동적이다.
- 예: RAG는 non-cached prefill 5,500 token, CAG는 500 token → CAG가 1/10 입력으로 TTFT 5초→0.5초 가능.
- 단 비용: 외부 vendor의 cached input은 regular의 10~25%(GPT-5: 0.125 vs 1.25). 예시 계산에서 CAG는 100k cached input 때문에 RAG($0.007)의 거의 2배($0.013) — TTFT는 빠르나 비용은 RAG가 저렴할 수 있음.

---

## 11. Self-Hosting — Offloading/Compression/Blending
> self-host 시 KV cache를 serving stack의 first-class citizen으로 저장·스케줄·이동·eviction한다(LMCache 등).
- KV cache offloading: GPU→CPU(~3배 공간), →SSD(~50배), →Redis/S3. 한 instance가 다수 long-context/tenant 캐싱 → replica·routing 감소로 최대 4배 비용 절감.
- KV cache compression: 전통 quantization 또는 CacheGen(분포 기반 bitstream 인코딩)으로 전송 가속·GPU 공간 확보.
- KV cache blending(CacheBlend): 서로 다른 RAG chunk의 KV cache를 병합. 단순 concat은 self-attention의 cross-token 관계 파괴 → 일부(기본 15%)만 선택적 재계산해 품질 유지·전체 재계산 회피.
- 벤치(Qwen3-14B): cold(hit 0)엔 LMCache가 오버헤드로 느림; warm 후 vanilla는 10번째 이후 eviction으로 prefill 재수행(6~7초)하나 LMCache는 CPU 검색(~1초). 고부하·다양한 prefix에서 LMCache throughput이 vanilla(~4,700 tok/s)의 ~16배.

---

## Summary (핵심 정리)
- speculative decoding(draft/self-drafting/n-gram)은 decode의 ITL을 가속하나 decode-only·acceptance·batch 특성에 따라 trade-off가 있다.
- 대형 모델은 DP(throughput)·TP(width)·PP(depth)·EP(MoE expert) 병렬화로 분산 서빙하며, NVLink/inter-node 통신 특성이 TP/PP 선택을 좌우한다.
- PD disaggregation은 prefill/decode를 분리해 TTFT·ITL·하드웨어·scaling을 독립 최적화하며, KV cache transfer 오버헤드를 compute 중첩으로 1% 미만까지 줄인다.
- advanced KV caching(long-context CAG·tiered offloading·compression·CacheBlend)은 TTFT를 크게 줄이며, 현대 serving은 kernel→execution engine→cache management→routing/orchestration의 통합 다층 stack이다.
