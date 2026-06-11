# 06. Essential LLM Optimization Techniques

## 챕터 개요 (3줄 요약)
- request batching·scheduling(dynamic → continuous → chunked prefill)으로 GPU idle을 줄이고 parallelism·throughput을 높인다.
- attention 최적화(MHA→MQA→GQA→MLA로 KV cache 축소, FlashAttention·PagedAttention 커널)와 model compression(quantization·distillation·pruning)으로 compute·memory를 절감한다.
- prefix caching(RadixAttention)은 이전 prompt의 KV cache를 재사용해 multiturn chat·long context에서 TTFT를 대폭 단축한다.

---

## 1. Why Batching in Real-Time Serving
> prefill은 토큰 병렬 처리로 compute-bound이고, decode는 token을 하나씩 생성해 memory bandwidth-bound이므로 batching은 특히 decode에서 효과적이다.
- 여러 request를 묶으면 weight를 한 번만 읽고 더 많은 token을 생성 → arithmetic intensity 인위적 증가, GPU FLOPS 활용↑.
- prefill은 prompt가 작지 않으면(>1,024 token) 이미 compute를 포화시켜 batching 효과 제한적.

---

## 2. Dynamic Batching
> dynamic batching은 추론 시점에 max batch size와 max delay time 두 파라미터로 incoming request를 묶어 latency·throughput을 동적으로 균형한다.
- client/static batching(꽉 채울 때까지 대기)은 offline엔 좋으나 online은 도착 간격이 커서 latency 문제.
- ferry-boat 비유: 10인승 보트(batch size 10) + 5분 대기(max delay) — 둘 중 먼저 충족 시 출발.
- 튜닝: latency SLA 내에서 batch size 최대화, max delay는 너무 길면 대기↑·너무 짧으면 batch 미달.

---

## 3. Continuous Batching (+ max number of batched tokens)
> continuous batching(inflight/iterative)은 고정 batch/시간 창을 기다리지 않고, 실행 중 request가 끝나는 즉시 대기 request를 batch에 추가해 가변 길이로 인한 GPU idle을 제거한다.
- LLM은 input/output 길이가 달라 dynamic batching에선 가장 긴 request가 전체 완료 시간을 결정 → idle 발생.
- ferry 비유: 10인승 1대 대신 1인승 10대 — 도착 즉시 출발, 낭비 없음.
- 파라미터: max batch size(request 수준 상한) + max number of batched tokens(token 수준 세밀 제어; prefill은 token 수, decode는 batch size가 주 제약). vLLM: --max-num-batched-tokens, --max-num-seqs.

---

## 4. Continuous Batching with Chunked Prefill
> chunked prefill은 긴 input prompt를 decode box와 비슷한 크기의 작은 chunk로 분할해, 긴 prefill이 decode를 막지 않게 하여 ITL을 개선한다.
- prefill 우선 시 실행 중 request가 idle, prefill+decode 함께 batching해도 decode가 prefill보다 빨라 지연 잔존.
- trade-off: ITL·throughput 개선 ↔ TTFT 증가·E2E latency 약간 악화(작은 prefill 오버헤드). chunk token 수는 중간값(오버헤드 vs GPU 포화) 튜닝.
- 더 advanced한 prefill-decode disaggregation은 ch7.

---

## 5. Scalable Attention Mechanisms (MHA/MQA/GQA/MLA)
> KV cache 축소는 decode의 memory bandwidth 압박 완화·batch size↑·long context를 가능케 해 serving 성능을 높인다.
- MHA: query마다 별도 K/V head → KV cache 최대·비효율.
- MQA: 모든 query가 단일 K/V head 공유 → 32~64배 축소이나 accuracy 크게 저하.
- GQA: query head를 그룹화해 그룹별 K/V 공유 → MHA·MQA의 좋은 절충(Llama3 config: num_key_value_heads=8).
- MLA(DeepSeek): KV 수를 줄이는 대신 영리하게 compress → "GQA 2.25 그룹 수준 cache로 MHA보다 강한 성능". 모두 model architecture 수준 선택.

---

## 6. Kernel Fusion, FlashAttention, PagedAttention
> 특화 GPU kernel은 GPU 활용·추론 속도·throughput을 크게 높인다.
- kernel fusion: 여러 연산(곱셈+덧셈)을 하나로 병합해 register/shared memory의 데이터를 재사용, global memory 왕복 제거.
- FlashAttention: attention이 memory bandwidth-bound임에 착안, 큰 행렬을 tiling해 SRAM/register에서 계산하고 HBM materialize 회피 + online softmax 융합. FA2/3은 GEMM·softmax 중첩으로 H100 활용 개선. (FlashInfer·xFormers·Triton 등도 존재; 백엔드가 기본 kernel 선택).
- PagedAttention(vLLM): OS paging처럼 KV cache를 고정 크기 block으로 나누고 block table로 매핑 → 비연속 저장·fragmentation 제거("20~38% 사용"→"near-zero waste"). continuous batching처럼 사실상 기본 기능.

---

## 7. Model Compression — Quantization (개념·포맷)
> quantization은 weight·activation·KV cache의 precision을 고비트→저비트로 낮춰 model 크기·data movement를 줄이고 연산을 가속한다(정확도 일부 희생).
- 오차: rounding error(표현 불가 값 반올림)와 clamping error(범위 초과 클램프) → 심한 왜곡 회피 위해 scaling 전략(symmetric/asymmetric) 사용.
- FP 포맷: FP32(1+8+23), FP16(1+5+10, 정밀도↑), BF16(1+8+7, FP32와 같은 지수 범위로 변환 용이·학습 적합).
- 정수형(INT8/INT4)은 균일 분포, 부동소수(FP8/FP4)는 0 근처 밀도↑ 비균일(logarithmic). 모델 파라미터 분포에 맞춰 선택.

---

## 8. How Quantization Helps & W4A16 vs W8A8
> quantization은 (1) 모델 크기 축소(GPU 적재·KV cache 공간 확보) (2) data movement 감소(decode latency↓) (3) 저비트 연산으로 FLOPS 2배(16→8bit)로 도움을 준다.
- weight-only(W4A16): 크기·data movement만 절감, 실행 시 dequant 필요로 연산 가속 없음(저배치·long generation·latency 민감에 유리). mixed-precision kernel(Marlin/Machete)로 dequant 회피 가능.
- weight+activation(W8A8): activation도 양자화해 compute-bound에서 FLOPS↑(long context·high throughput·high batch). dynamic(정확)/static(빠름) scaling 선택.
- 생산 주류: weight-only는 GPTQ/AWQ W4A16, weight+act는 W8A8(INT8→FP8 E4M3). FP8은 Hopper/Blackwell만 지원.
- 벤치(Qwen2.5-7B): 저배치 W4A16 ~300%·FP8 ~150% 개선, 고배치는 W8A8가 우세(W4A16은 dequant 오버헤드로 TTFT 악화).

---

## 9. KV Cache/GGUF Quantization, Accuracy, QAT
> attention·KV cache quantization, GGUF, 정확도 trade-off, QAT 등 추가 기법.
- KV cache quantization: GPU memory 확보·batch↑·prefix caching 이점이나, attention이 고정밀이면 dequant 필요로 latency 개선 미미 → quantized attention kernel과 병행해야 효과.
- GGUF: CPU/Apple Silicon용 portable·저자원 배포 포맷(Llama.cpp).
- 정확도: GPTQ W4A16·AWQ·FP8은 실측 손실 미미. 큰 모델일수록 weight·KV cache quantization에 민감. LM Eval로 평가.
- PTQ(post-training; 쉽고 주류) vs QAT(학습 중 fake quant; 4bit 이하 공격적 압축에 필요·정확도 우수하나 비용·유연성↓). 예 GPT-OSS는 MoE layer를 MXFP4 QAT로 압축(120b→58.5GB, 20b→10.5GB).

---

## 10. Distillation & Pruning
> distillation은 큰 teacher 모델의 지식(hard label·logits·loss)을 작은 student 모델로 전이해 새 소형 모델을 학습한다 — latency·throughput 개선 잠재력이 가장 크다.
- 예: DeepSeek R1 671B → Llama/Qwen 기반 1.5~70B distilled. 70B 모델이 MATH-500 94.5 등 근접 성능. teacher 모델 full access 필요.
- 가이드: distilled 모델이 있으면 평가 후 채택+quantization, 없으면 비용·정확도 손실 때문에 quantization 우선.
- pruning(최소 인기): 과파라미터 제거 — structured(섹션 제거)/unstructured(개별 weight)/2:4 semi-structured sparsity(4중 2 제거). Sparse Llama 3.1은 98% 정확도 회복·30% throughput↑(Ampere/Hopper sparse Tensor Core로 50% sparsity가 matmul 2배 가속).

---

## 11. Prefix Caching & RadixAttention
> prefix caching는 전체 prompt 대신 prompt의 "prefix"를 이전 처리분과 매칭해, 일치하는 부분의 KV cache 재계산을 생략하고 GPU memory에서 재사용한다(LRU eviction).
- 일반 request caching은 자유 형식 텍스트라 hit rate 낮음 → prefix 매칭이 효과적.
- RadixAttention(SGLang): radix tree(trie)로 prefix string 추적, 노드가 GPU의 KV cache에 매핑, LRU로 leaf eviction.
- 적용 시나리오: (1) multiturn chat(이전 대화 누적 → 재사용으로 TTFT 단축) (2) long context serving(긴 context KV cache 재사용). hit rate 5%여도 가치 있어 거의 기본 활성.

---

## 12. Prefix Cache Best Practices & Scaling
> hit rate 향상이 핵심 — prompt를 static 부분(앞 prefix)과 dynamic 부분(뒤 user input)으로 명확히 분리·일관 포맷팅한다.
- "Document"→"Documents" 한 글자, 공백·순서 차이도 cache miss 유발 → 프로그램적·일관 조립. RAG는 정렬·dedup로 최장 prefix hit 추구.
- scaling: 단일 인스턴스는 KV cache용 GPU memory 충분히 확보. horizontal scaling 시 consistent hashing 기반 cache-aware routing으로 prefix가 저장된 인스턴스에 라우팅.
- GPU memory 부족 시 KV cache를 CPU/SSD offload(ch7). multi-tenant는 user/session ID를 prompt에 주입해 prefix 격리(타 고객 데이터 추론 방지).

---

## Summary (핵심 정리)
- continuous batching(+chunked prefill)은 가변 길이 LLM 요청의 GPU idle을 제거하는 online serving 표준이며, TTFT·ITL·throughput을 SLA에 맞춰 튜닝한다.
- attention 최적화는 KV cache 축소(MHA→MQA→GQA→MLA)와 커널(kernel fusion·FlashAttention의 tiling/SRAM·PagedAttention의 paged KV cache)로 compute·memory를 절감한다.
- model compression은 정확도를 일부 내주고 크기·연산을 줄이며, 주류는 PTQ quantization(W4A16 GPTQ/AWQ, W8A8 FP8)이고 distillation·pruning은 보조적이다.
- prefix caching(RadixAttention)은 storage를 compute와 교환해 multiturn chat·long context에서 TTFT를 크게 줄이며, 일관된 prompt 구조와 cache-aware routing이 hit rate를 좌우한다.
