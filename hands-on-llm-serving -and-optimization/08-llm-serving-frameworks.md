# 08. LLM Serving Frameworks

## 챕터 개요 (3줄 요약)
- 범용 ML serving(TF Serving·TorchServe·Triton)은 autoregressive·long context·continuous batching·streaming 요구를 충족 못 해 LLM 전용 framework가 필요하다.
- vLLM은 paged KV caching·continuous batching을 핵심으로 LLMEngine·Scheduler·ModelExecutor 구조와 token-level scheduling·layered optimization 전략을 갖춘다.
- TensorRT-LLM·SGLang·llama.cpp는 각각 NVIDIA 최고 효율·agentic/structured/multi-vendor·local/edge에 특화되며, SLO·하드웨어 기준으로 선택한다.

---

## 1. Why Specialized LLM Serving Frameworks
> 범용 framework는 짧은 입력·고정 shape·예측 가능한 latency용으로 설계돼 LLM의 고유 과제를 다루지 못한다.
- LLM 과제: autoregressive 생성(초~분 단위 세션), context length 폭발(KV cache 병목), continuous batching(가변 길이), streaming(TTFT 수백ms), GPU 자원 활용(fragmentation·idle 불허).
- vLLM·TensorRT-LLM·SGLang은 paged KV caching·continuous batching·LLM quantization·speculative decoding으로 대응 → 주류 선택.

---

## 2. vLLM 개요 & Architecture
> vLLM은 paged KV caching + continuous batching으로 throughput·latency를 개선하고, quantization·speculative decoding·streaming·multi-GPU를 지원하는 가장 인기 있는 framework다.
- "just works" — battle-tested, open source/fine-tuned 모델 통합 용이, 깊은 튜닝 없이 GPU 효율 극대화, 확장 가능한 clean 아키텍처.
- 사용 2가지: LLM class(in-process library, offline inference), API Server(OpenAI-compatible HTTP, multi-client·streaming).
- 핵심 컴포넌트: LLMEngine(공개 API·request lifecycle), EngineCore(중앙 오케스트레이터·inner loop), Scheduler(traffic controller).

---

## 3. ModelExecutor / GPUWorker / GPUModelRunner & 초기화
> vLLM은 모델을 별도 process(group)에서 실행해 layered 구조로 cross-process 통신과 forward-pass를 처리한다.
- ModelExecutor(다수 worker process 오케스트레이션) → GPUWorker(worker process 인터페이스·device/model lifecycle) → GPUModelRunner(실제 신경망 실행).
- 초기화 4단계: (1) main process에서 컴포넌트 초기화·config 분배 (2) MultiProcessExecutor가 worker process 생성 + rpc_broadcast_mq (3) 각 GPUWorker가 CUDA·통신·모델 로드 (4) GPUModelRunner가 registry에서 구현 찾아 weight 적재. backend는 mp(단일 node) 또는 ray(cross-machine).

---

## 4. Generation-Request Execution Workflow
> llm.generate 호출 시: Processor가 입력 검증·tokenize → LLMEngine이 EngineCore 반복 호출 → Scheduler가 다음 batch·token 결정(paged attention·continuous batching) → MultiProcessExecutor가 worker에 위임해 forward pass → output processor가 최종 응답 생성.
- 모델 실행·model-specific 최적화는 GPUWorker, 일반 serving 계획·최적화는 Scheduler가 담당.

---

## 5. Scheduler Deep Dive
> Scheduler는 inference pipeline의 traffic controller로, GPU memory·KV cache block·token budget을 경쟁 요청에 효율 배분한다.
- 역할: request resource orchestration(WAITING/RUNNING queue), token-level scheduling(request 단위가 아닌 token 단위 세밀 제어), optimization integration hub(prefix caching·speculative decoding·chunked prefill·distributed KV transfer), dynamic load balancing, request lifecycle(FCFS/priority, preemption).
- 워크플로: schedule state 구축 → running 요청 우선 처리(chunked prefill·prefix caching·speculative decoding 적용) → waiting queue 활성화 → post-processing(LoRA·encoder·draft token) → SchedulerOutput 조립.
- 구현: num_computed_tokens와 num_tokens_with_spec 간 gap 최소화가 최적화 chain 구동. 요청 우선순위(queue)와 token-level scheduling(gap 비교)을 명확히 분리.

---

## 6. vLLM Layered Optimization Strategy
> 최적화는 적절한 level에서 이뤄져야 한다 — LLM은 빠르게 진화하므로 단일 모델/하드웨어에 hardcode 불가.
- Scheduler: system-wide·model-agnostic(fairness·efficiency·scalability).
- ModelExecutor: model architecture-specific(fused attention kernel 등).
- Model layer: component-specific(KV cache reuse·flash attention·operator fusion).
- CustomOp: hardware-specific(CUDA kernel·tensor core·quantized operator).
- 이 분리로 Scheduler를 모델 디테일로 어지럽히지 않으면서 깊은 특화 가능 → 새 architecture·하드웨어를 적절 layer에 슬롯인하는 future-proof 설계.

---

## 7. TensorRT-LLM
> NVIDIA의 고성능 LLM inference 라이브러리로, 모델 checkpoint에서 튜닝된 TRT engine을 빌드하고 Python/C++ runtime을 제공한다.
- 기능: in-flight batching(continuous batching), paged KV cache, speculative decoding, 다중 precision quantization(FP8/FP4/INT4/INT8), TP/PP. Dynamo·Triton과 긴밀 통합.
- 목표: NVIDIA GPU의 최대 실용 성능(Tensor Core·CUDA kernel 활용) 시연. NVIDIA 하드웨어·스택 표준화 조직에 최적(최고 효율·throughput).

---

## 8. SGLang
> structured-generation·agent 애플리케이션을 겨냥한 고성능 LLM/VLM serving framework로, 빠른 backend runtime과 유연한 frontend 언어/API를 co-design한다.
- 기능: RadixAttention(prefix/KV 재사용), continuous batching, paged attention, speculative decoding(EAGLE-2/3), chunked prefill, structured output(JSON/regex/EBNF), multi-LoRA batching, TP/PP/EP/DP.
- 하드웨어: NVIDIA·AMD·CPU·TPU·Jetson·Ascend 등 multi-vendor. vLLM의 peer로 portability·agentic 강점이나 vLLM이 더 큰 커뮤니티.

---

## 9. Llama.cpp
> 거의 모든 머신(laptop~edge)에서 open-weight LLM을 효율 실행하는 경량 C/C++ stack으로, GGUF 포맷·OpenAI-compatible HTTP 서버를 제공한다.
- 특징: 최소 의존성·빠른 startup·공격적 정수 quantization(8/6/5/4-bit)·portable CPU/GPU backend(SIMD·Metal·CUDA/ROCm·Vulkan).
- "runs anywhere" 철학 — peak throughput보다 portability·단순성·비용 효율 우선. local 개발, private/on-prem assistant(데이터 경계 유지), edge inference에 적합. Ollama로 REST API 래핑 가능.
- local 추론은 목표가 다름: latency·responsiveness 우선, 저동시성, footprint·cost 제약, privacy·offline 우선.

---

## 10. Selecting the Right Framework
> 가장 화려한 벤치마크가 아니라 SLO·하드웨어·운영 현실에 맞는 framework를 선택한다(workload first).
- 평가법: SLO부터 정의(TTFT·p95/p99·TPS·cost·품질) → 실제 prompt 분석(prefill/decode-heavy·context·tool call) → 동일 조건 apples-to-apples 비교 → operability(cold-start·observability·autoscaling·multi-tenancy) 측정 → 하드웨어·vendor lock-in 고려 → 변화 대비.
- 권장: 빠른 production·broad 모델·Python은 vLLM / agentic·strict JSON·multi-vendor는 SGLang / NVIDIA all-in·peak tokens/dollar는 TensorRT-LLM / local·edge·tiny footprint·privacy는 llama.cpp.
- 3~6개월마다 재평가, framework abstraction layer + exit plan으로 앱 재작성 없이 교체 가능하게.

---

## Summary (핵심 정리)
- 범용 ML serving은 token-level scheduling·KV cache 관리·long-context·streaming-first 실행이 없어 LLM에 부족하다.
- vLLM은 LLMEngine·Scheduler·ModelExecutor 구조와 token-level scheduling, system→model→layer→hardware의 4단 layered optimization으로 동작한다.
- TensorRT-LLM(NVIDIA 최고 효율), llama.cpp(경량 local/edge), SGLang(agentic·structured·multi-vendor)이 vLLM을 보완한다.
- framework 선택은 맥락적이며, portable 인터페이스·주기적 재평가로 framework-agnostic을 유지하는 것이 핵심이다.
