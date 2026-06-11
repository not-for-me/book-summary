# 05. Challenges When Serving LLMs

## 챕터 개요 (3줄 요약)
- LLM serving 최적화는 customer experience(latency), cost efficiency(inference가 최대 비용), scalability/peak load 측면에서 비즈니스 성패를 좌우한다.
- GPU 핵심 스펙(compute FLOPS·memory 용량·memory bandwidth·interconnect)과 model loading(weight + KV cache) 메모리 산정이 serving 가능성을 결정한다.
- arithmetic intensity와 roofline model로 분석하면 LLM serving은 prefill(compute-bound 가능)과 decode(항상 memory bandwidth-bound)로 나뉜다.

---

## 1. Why Optimizing LLM Serving is Important
> LLM 최적화는 customer experience, cost efficiency, scalability/peak load/feasibility 세 측면에서 중요하다.
- customer experience: latency와 만족도는 역상관 — 20초→1초는 게임체인저이나 0.1초→0.01초는 인지 불가(이땐 latency를 throughput과 교환). 또한 같은 latency로 더 큰 모델(8B→70B) 서빙 시 품질 향상.
- cost efficiency: inference가 training보다 큰 비용 — 학습은 선투자이나 inference는 사용량에 따라 지속 누적, agent·복잡 workflow가 multiple LLM call로 비용 증폭. 최적화로 동일 HW에서 throughput↑·저급 칩 사용으로 비용 절감.
- scalability: Black Friday 400% 급증 같은 peak 대응, 고급 GPU 부족 지역에서 저급 칩으로도 운영 가능한 유연성.

---

## 2. The Role of Accelerator Chips — Reading GPU Specs
> GPU 선택은 memory 용량·compute·bandwidth·interconnect를 결정하므로 serving의 가장 중요한 의사결정 중 하나다.
- compute(teraFLOPS): 정밀도별 이론 연산 능력 — 낮은 precision(FP16→FP8)일수록 거의 2배 빠름.
- memory(VRAM, HBM): 모델 적재 용량 / memory bandwidth: 연산용 데이터 전송 속도. H100 SXM(80GB, 3.35TB/s, 더 빠른 compute) vs H100 NVL(94GB, 3.9TB/s, 더 큰 memory).
- 피자 가게 비유: oven=compute(FLOPS), 재료 준비 속도=memory bandwidth, 냉장고 용량=memory capacity. 셋의 균형이 핵심.

---

## 3. GPU Interconnects (Intra/Inter-node) & Power
> interconnect는 multi-GPU 간 고속 데이터 전송으로, 대형 모델·latency 민감 use case에서 model serving에도 중요해졌다.
- intra-node: PCIe(128 GB/s, 최저가) < NVLink Bridge(600 GB/s, 2 GPU만) < NVLink/NVSwitch(900 GB/s, 최대 8 GPU all-to-all). SXM은 motherboard 직결로 PCIe보다 우수.
- inter-node: node당 최대 8 GPU 한계 → 초과 시 모델을 여러 node에 shard. InfiniBand + GPUDirect RDMA로 ~50 GB/s(intra-node보다 훨씬 느림). 대부분 배포는 단일 node 1~8 GPU replica를 horizontal scaling.
- power(TDP, W): cloud 사용자는 추상화되나 datacenter는 first-class 제약(performance/watt), edge는 전체 시스템 정의 제약.

---

## 4. Comparing Popular GPUs
> 더 강력한 GPU는 항상 비싸므로, 서빙할 모델이 그 성능·기능의 이점을 받는지에 따라 선택한다.
- small model(Llama-3-8B, latency 비중요): A10(24GB, 최저가·가용성 높음).
- mid-sized(DeepSeek-R1-Distill-Qwen-14B): L40S(48GB, FP8 지원, multi-GPU 오버헤드 회피).
- large(DeepSeek-R1 671B MoE): NVLink Switch 필수, 8×H200 FP8 구성, expert를 여러 GPU/node에 분산.
- 기술은 빠르게 바뀌지만 기초 개념과 옵션 비교 직관은 변하지 않음.

---

## 5. Bottlenecks in LLM Model Loading
> 효율적 serving의 첫 단계는 model weight를 GPU memory에 로드·캐싱하는 것 — disk/CPU memory는 전송 속도가 느려 실시간 추론에 부적합하다.
- 데이터 흐름: weight → CPU(system) memory → GPU memory에 캐싱. bandwidth: SSD(0.5~14 GB/s) < CPU DRAM(50~200 GB/s) < GPU HBM(300 GB/s~3 TB/s).
- CPU memory(DRAM, 용량 최적화) vs GPU memory(HBM, 병렬 전송 최적화) — weight는 GPU memory에 있어야 compute 활용.

---

## 6. Estimating Model & KV Cache Size
> 모델 메모리는 parameter 수 × data type(precision) 크기로 산정한다.
- precision: FP32(4B), FP16/BF16(2B), INT8/FP8(1B). 낮출수록 정확도 일부 희생·크기↓·성능↑.
- 예: Llama-2-7b ≈ 7B × 2B = 14GB.
- KV cache per token = 2 × layers × heads × head_dim × dtype_size (예 Llama-2-7b: 2×32×32×128×2 = 0.5MB/token).
- Total KV cache = per-token × (max batch × max seq len). 예: 0.5MB × 4096 × 16 = 32GB (모델 14GB보다 큼!).
- A10(24GB)은 4 요청, L40S(48GB)는 16 요청 동시 처리 → L40S가 더 비싸도 cost-efficient. 권장: 모델 크기의 ~2배 GPU memory 확보(parallelism·OOM 방지).

---

## 7. Bottlenecks in LLM Model Execution — Compute vs Memory Bandwidth
> serving이 compute FLOPS에 묶이는지 memory bandwidth에 묶이는지는 arithmetic intensity(FLOPS / data movement)로 판단한다.
- data movement는 실행 시 weight를 HBM(off-chip, 느림) → SRAM(L2/L1/shared, 빠름·작음) → register로 옮기는 것. GPU memory bandwidth가 병목 수치.
- roofline model: arithmetic intensity가 crossover(L40S 기준 362 TFLOPS/864 GB/s ≈ 419 FLOPS/B) 미만이면 memory bandwidth-bound, 초과면 compute-bound.

---

## 8. Arithmetic Intensity in Matrix Multiplications & Prefill/Decode
> Transformer의 self-attention·feedforward는 대부분 matmul이며, matrix가 충분히 크면 compute-bound가 된다.
- matmul arithmetic intensity = M×N×K / (M×K + K×N + M×N). M=N=K일 때 64→21, 512→170(둘 다 memory-bound), 4096→1365(compute-bound).
- 입력 tensor [batch, seq len, hidden dim]; batch=1이면 [seq len, hidden dim] matrix.
- prefill: 모든 token 동시 처리 → s가 큼 → seq len 길면 compute-bound. decode: token 1개씩(s=1) → arithmetic intensity ≈ 0.5로 항상 memory bandwidth-bound.
- 결론: compute-bound면 FLOPS·연산 최적화, memory-bound면 불필요한 data movement 최소화.

---

## 9. Other AI Accelerators and Trends
> NVIDIA GPGPU가 여전히 지배적이나(CUDA 생태계 성숙, precision·bandwidth·interconnect 유연성), AMD MI300X·Intel Gaudi2·Google TPU·AWS Inferentia·Huawei Ascend 등 경쟁.
- "memory wall": compute는 빠르게 발전했으나 memory/inter-GPU bandwidth는 뒤처져 data movement가 병목.
- 대응 trend 1: on-chip SRAM에 연산을 가까이(Groq 등) → 극저 latency, 단 비용·model partitioning 부담.
- 대응 trend 2: 긴밀 결합 multi-GPU 시스템 — NVIDIA GB200 NVL72(Grace CPU 36 + Blackwell GPU 72, NVLink-C2C/Switch), 대형 MoE disaggregated serving에 강력.

---

## Summary (핵심 정리)
- 효율적 LLM serving은 customer experience·cost·scalability를 좌우하며, inference 비용이 training을 넘어서는 추세다.
- GPU 핵심 스펙은 memory size·compute FLOPS·memory bandwidth이며, 대형 모델은 intra/inter-node interconnect가 중요하다.
- model loading은 weight + KV cache 공간을 모두 확보해야 하고(권장 모델 크기의 ~2배), KV cache는 batch·seq len에 따라 모델보다 커질 수 있다.
- arithmetic intensity·roofline 분석으로 LLM serving은 prefill(compute-bound 가능)과 decode(항상 memory bandwidth-bound)로 구분되며, 이 직관이 후속 최적화 기법 선택의 기반이다.
