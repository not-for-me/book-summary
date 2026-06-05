# 09. Inference Optimization

## 챕터 개요 (3줄 요약)
- 모델을 더 빠르고 저렴하게 만드는 inference 최적화로, model·hardware·service 수준에서 이뤄지는 학제간 분야다.
- inference의 병목(compute-bound vs memory bandwidth-bound), 성능 지표(latency·throughput·utilization), AI accelerator를 다룬다.
- model 수준(compression, decoding 병목 극복, attention 최적화)과 service 수준(batching, caching, parallelism) 최적화 기법을 설명한다.

---

## 1. Understanding Inference Optimization
> inference는 입력에 대해 출력을 계산하는 과정으로, 병목을 찾아 해결한다.

### Computational Bottlenecks
> compute-bound(연산량 제한)와 memory bandwidth-bound(데이터 전송 속도 제한)로 나뉜다.

arithmetic intensity(메모리 접근 바이트당 연산 수)로 분류하며 roofline chart로 시각화한다. 이미지 생성은 compute-bound, autoregressive LM은 memory bandwidth-bound 경향. transformer inference의 prefill(입력 병렬 처리)은 compute-bound, decode(토큰 순차 생성)는 memory bandwidth-bound라 production에서 분리한다.

### Online/Batch APIs & Metrics
> online API는 latency, batch API는 cost를 최적화한다.

batch API는 ~50% 저렴하나 처리 시간이 길다(synthetic 데이터 생성, 주기적 리포트 등에 적합). streaming mode는 첫 토큰 대기를 줄인다.

**Latency**: TTFT(Time To First Token, prefill 단계), TPOT(Time Per Output Token, decode 단계). total = TTFT + TPOT × 출력 토큰 수. percentile(p50/p90/p99)로 봐야 한다.

**Throughput/Goodput**: 초당 출력 토큰(TPS), RPS/RPM. cost와 직결. latency/throughput trade-off가 있어 SLO를 만족하는 goodput을 본다.

**Utilization**: nvidia-smi의 GPU utilization은 오해 소지. MFU(Model FLOP/s Utilization, 실제/이론 throughput)와 MBU(Model Bandwidth Utilization)가 유용하다.

```
MBU = (param수 × bytes/param × tokens/s) / 이론 bandwidth
total latency = TTFT + TPOT × (출력 토큰 수)
```

### AI Accelerators
> AI 워크로드 가속용 칩으로, GPU가 지배적이다.

CPU(소수의 강력한 코어, 범용)와 달리 GPU(수천 개 작은 코어, 병렬 처리). inference 전용 칩이 부상(저정밀도·빠른 메모리 접근 최적화). 주요 특성: 연산 능력(FLOP/s, 정밀도 낮을수록 많은 연산), 메모리 크기·bandwidth(CPU DRAM < GPU HBM < on-chip SRAM 계층), 전력 소비(TDP). 워크로드가 compute-bound면 FLOP/s, memory-bound면 bandwidth·메모리가 중요.

---

## 2. Model Optimization
> 모델 자체를 수정해 효율화하며, 동작이 바뀔 수 있다.

### Model Compression
> 모델 크기를 줄여 빠르게 만든다.

quantization(정밀도 감소, 가장 인기), distillation(작은 모델이 큰 모델 모방), pruning(노드 제거 또는 파라미터를 0으로 만들어 sparse화). pruning은 효과적이나 어렵고 sparse 하드웨어 지원이 필요하다.

### Overcoming Autoregressive Decoding Bottleneck
> 토큰 순차 생성의 병목을 극복한다.

**Speculative decoding**: 빠른 draft 모델이 K토큰 생성 → target 모델이 병렬 검증·수락(품질 불변, 검증이 생성보다 빠름). **Inference with reference**: 입력에서 draft 토큰 선택(retrieval·코딩·multi-turn에서 중복 많을 때). **Parallel decoding**: 순차 의존성 깨고 여러 토큰 동시 생성 후 검증(Lookahead-Jacobi, Medusa-tree attention).

### Attention Mechanism Optimization
> KV cache로 이전 토큰의 key/value 벡터를 재사용한다.

KV cache는 inference에만 쓰이며 sequence length·batch size에 비례해 커진다(장문 context의 병목). 세 갈래: **redesign**(local windowed attention, multi-query/grouped-query/cross-layer attention으로 KV 감소; 학습 시에만 적용), **KV cache 최적화**(PagedAttention, quantization, compression), **kernel 작성**(FlashAttention).

```
KV cache 메모리 = 2 × B × S × L × H × M
(batch, seq len, layers, model dim, bytes)
```

### Kernels and Compilers
> 특정 하드웨어에 최적화된 코드(kernel)와 lowering 도구(compiler).

CUDA, Triton, ROCm로 작성. 기법: vectorization, parallelization, loop tiling, operator fusion. compiler(torch.compile, XLA, TensorRT)가 모델을 하드웨어 kernel로 변환(lowering)한다.

---

## 3. Inference Service Optimization
> 자원 관리에 집중하며, 모델을 수정하지 않아 출력 품질이 변하지 않는다.

### Batching
> 동시 도착 요청을 묶어 throughput을 높인다.

**Static batching**(고정 크기, 마지막 요청까지 대기), **Dynamic batching**(최대 시간 창; latency 관리), **Continuous batching**(완료 응답 즉시 반환, 빈자리에 새 요청 추가; in-flight batching).

### Decoupling Prefill/Decode & Prompt Caching
> prefill(compute-bound)과 decode(memory-bound)를 다른 인스턴스에 분리한다.

같은 머신에서 처리하면 자원 경쟁으로 TTFT·TPOT 저하. 분리하면 처리량 향상(prefill:decode 비율은 워크로드·latency 우선순위에 따라). **Prompt caching**(context/prefix cache): 중복 세그먼트(system prompt, 긴 문서, 대화 이력)를 캐싱·재사용해 latency·cost 대폭 절감.

### Parallelism
> accelerator의 병렬 처리를 활용한다.

**Replica parallelism**(모델 복제, 구현 간단). **Model parallelism**: tensor parallelism(operator 내 tensor 분할, latency↓·대형 모델 서빙), pipeline parallelism(단계별 분할, 통신 overhead로 latency↑이나 training throughput↑). **Context/sequence parallelism**(장문 입력 처리 효율화).

---

## Summary (핵심 정리)
- inference의 cost·latency가 모델 usability를 좌우하며, latency(TTFT·TPOT)·throughput·utilization(MFU·MBU)으로 효율을 측정한다(latency/throughput trade-off 존재).
- model 수준 최적화(quantization, distillation, decoding 병목 극복, attention/KV cache 최적화)는 모델을 수정해 동작이 바뀔 수 있다.
- service 수준 최적화(batching, prefill/decode 분리, prompt caching, parallelism)는 모델을 그대로 두고 서빙 방식만 바꾸며, 가장 임팩트 큰 기법은 quantization·tensor parallelism·replica parallelism·attention 최적화다.
