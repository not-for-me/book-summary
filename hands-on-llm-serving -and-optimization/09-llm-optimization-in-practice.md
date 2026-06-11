# 09. LLM Optimization in Practice

## 챕터 개요 (3줄 요약)
- Qwen3-14B + vLLM로 실제 serving 최적화 과정을 단계별로 실습한다 — 하드웨어 점검, 벤치 트래픽 생성, metric 정의, 서빙·벤치마크·분석.
- model quantization(AWQ 4-bit)이 KV cache 공간을 확보해 throughput을 2.7배 높이고 TTFT를 42% 개선하는 등 핵심 효과를 측정으로 입증한다.
- distributed serving은 NVLink 유무에 따라 효과가 갈리며, latency·모델 크기 한계 극복이 진짜 가치이고 대부분은 horizontal scaling이 낫다.

---

## 1. LLM Serving Optimization Plan
> 목표는 Qwen3-14B 단일 instance의 token throughput 극대화 — 처리 token이 비용 기반이므로 throughput↑가 cost↓로 직결된다.
- throughput vs latency trade-off: throughput은 batching/queuing으로 자원 활용하나 대기로 per-request latency 증가. 보통 latency를 허용 범위 내로 유지하며 throughput 우선.
- 8단계 계획: (1) 하드웨어 점검 (2) 벤치 트래픽 생성 (3) metric 정의 (4) vLLM 서빙 셋업 (5) baseline 벤치 (6) quantized 벤치 (7) 추가 최적화 기법 (8) distributed serving 벤치.

---

## 2. Step 1 — Examine GPU Hardware
> nvidia-smi로 GPU·memory·utilization·driver/CUDA 버전을 점검해 벤치 결과 해석의 기반을 만든다(예: AWS g6e.2xlarge, 단일 L40S 46GB).
- 핵심 확인: CUDA/driver 호환성, performance state(P8 idle/P0-P1 full), power·utilization(저활용=batching/scheduling 비효율, 고전력 저throughput=memory/kernel 병목), memory usage.
- --query-gpu 플래그로 원하는 속성만 출력(L40S: compute_cap 8.9, 46068MiB).

---

## 3. Step 2 — Generate Benchmark Traffic
> 데이터셋 선택이 핵심 — 벤치는 배포 모델의 예상 사용 패턴(prefill-heavy vs decode-heavy)을 정확히 반영해야 한다.
- ShareGPT: 실제 대화 데이터, 다양한 prompt·응답 → 현실적 interactive 트래픽 평가.
- Prefix Repetition: 공통 prefix + 반복 suffix 합성 데이터 → repetition bias·cache 활용 평가(prefix 수 줄이면 반복↑).
- inspect_dataset.py로 prompt/output 길이 분포·histogram 확인. vllm bench serve CLI로 prompt 수·request rate·burstiness·concurrency 제어하며 트래픽 생성·metric 수집.

---

## 4. Step 3 — Define Evaluation Metrics
> throughput·latency·resource·workload·reliability 범주에서 핵심 metric을 정한다.
- 본 실습 4개: total token throughput(TPS, 시스템 효율 지표), output token throughput(TPS, decode 성능·비용 핵심), mean TTFT(ms, prefill 효율), mean ITL(ms, streaming UX).

---

## 5. Step 4-5 — Setup & Baseline Benchmark
> vLLM 기본 설정으로 Qwen3-14B를 호스팅해 baseline을 만든다.
- 메모리: 모델 27.5GB + KV cache 11GB(72,064 token) = 38.5/46GB. 모델이 65%+ 차지해 KV cache 공간 부족 → batch·concurrency 제약, eviction·recompute 증가로 throughput↓.
- ShareGPT 벤치(2,000 prompt, 10 RPS): total 474 TPS, TTFT 104ms, ITL 43ms.
- Prefix Repetition 벤치: total 1,123 TPS(반복 트래픽에 prefix caching·continuous batching·memory block sharing 자동 적용으로 throughput↑), GPU utilization 97%.

---

## 6. Step 6 — Quantized Model Benchmark
> AWQ 4-bit 양자화(Qwen3-14B-AWQ)는 weight footprint를 27.5GB→9.6GB로 줄여 17GB를 KV cache로 확보한다.
- KV cache token 72,064→191,056(2배+) → prefix recompute 감소·batch 확대.
- ShareGPT: total throughput 474→1,280 TPS(2.7배), TTFT 103.61→59.29ms(~42% 개선). Prefix Repetition도 유사 개선.
- 이 실험은 weight quantization의 data movement·memory 절감을 보여줌; compute 절감(activation quantization)은 ch6 참조.

---

## 7. Step 7 — Additional Optimization Techniques
> quantized 모델이 일반적으로 강력하나, 트래픽/GPU 특화 기법과 vLLM config 튜닝으로 한계를 더 밀 수 있다.
- long-context prefill-heavy: LMCache로 반복 prefix의 KV 재사용(multiturn·RAG).
- decode-heavy(긴 출력): speculative decoding으로 draft 모델이 다중 token 예측, decode iteration 감소.
- config knob: --gpu-memory-utilization 0.9, --max-model-len, --block-size 16(작을수록 cache 활용↑), --max-num-seqs, --max-num-batched-tokens, --enable-prefix-caching, --enable-chunked-prefill.
- 주의: overtune 금지 — 특정 GPU/workload에 overfit하면 portability·유지보수 악화. 현대 framework는 startup 시 자동 추론하므로 "어떤 기법을 적용할지"에 노력 집중.

---

## 8. Step 8 — Distributed Serving Benchmark
> 단일 node multi-GPU 분산 서빙을 벤치(non-hyperscale 일반 시나리오). vllm serve --tensor-parallel-size N으로 간단 구성.
- g6e(4×L40S): 단일 GPU가 throughput·TTFT 모두 최고 — multi-GPU가 더 나쁨.
- p4d(8×A100): 반대로 multi-GPU가 우수, 4-GPU가 최고 throughput·최저 latency.
- 원인: A100은 NVLink로 고속·저지연 통신 → tensor parallelism 효율적. L40S는 NVLink 없이 PCIe만 → inter-GPU 통신 오버헤드로 단일 GPU가 빠름.
- distributed serving이 항상 좋은 건 아님 — p4d에서도 4개 독립 instance가 4-GPU 단일 분산 모델보다 ~3배 throughput(9,816 vs 3,926 TPS). 진짜 가치는 vertical scaling(latency↓, 모델 크기 한계 극복; p4d TTFT 66→33ms ~50% 감소, horizontal scaling으론 불가).

---

## 9. Common Optimization Trade-offs
> 어떤 기법을 적용할지 결정에 "완벽한 config" 추구보다 더 많은 노력을 들인다 — 5가지 trade-off.
- throughput vs latency: batching은 throughput↑·latency↑(interactive는 latency, offline은 throughput).
- memory efficiency vs model quality: quantization은 memory↓·KV cache↑이나 정확도 일부 손실(4bit vs 8bit 균형).
- hardware utilization vs flexibility: 공격적 튜닝은 특정 GPU 최적화이나 일반화 안 됨 → flexibility 위해 약간의 효율 희생.
- vertical vs horizontal scaling: vertical(분산)은 대형 모델·latency↓, horizontal(다중 replica)은 throughput·fault tolerance↑. 대부분 production은 horizontal 중심.
- static vs adaptive serving: 정적 config는 예측 가능하나 overfit, adaptive runtime은 live 트래픽 기반 동적 조정(차세대 self-optimizing).

---

## Summary (핵심 정리)
- 최적화는 보편적 "best" config 찾기가 아니라 dominant bottleneck을 파악해 하드웨어·모델·scheduling의 적절한 level에서 기법을 적용하는 것이다.
- scenario·트래픽·목표 정의 → 대표 데이터셋·metric 준비 → 일반 throughput 최적화 baseline → workload 특화 기법(LMCache·speculative decoding) 순으로 진행한다.
- quantization은 memory·KV cache 확보로 throughput을 크게 높이며, distributed serving은 NVLink 유무·모델 크기·latency 요구에 따라 선택적으로 사용한다(대부분 horizontal scaling이 단순·효율적).
- overfit한 정적 config를 피하고, 최적화는 실제 사용자 시나리오 기반의 지속적 실험·측정·trade-off 분석으로 반복한다.
