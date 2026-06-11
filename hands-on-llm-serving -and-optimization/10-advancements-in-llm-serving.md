# 10. Advancements in LLM Serving

## 챕터 개요 (3줄 요약)
- semantic caching/routing은 embedding·vector search로 의도가 같은 prompt를 인식해 endpoint 수준에서 cache hit·모델 선택·tool filtering을 수행한다.
- multimodal serving·edge AI·multi-LoRA는 serving을 text 너머 vision·on-device·개인화 fine-tuned 모델로 확장한다.
- LLM serving 엔진은 RLHF의 sample 생성 단계를 떠받치며, 여기서는 throughput뿐 아니라 deterministic inference가 중요하다.

---

## 1. Semantic Caching & Routing
> serving 시스템이 semantic-aware해져, 정확한 prompt가 아닌 embedding·vector search로 의도가 같은 prompt를 인식해 endpoint 수준 라우팅을 한다.
- 효용: (1) 유사 prompt(같은 질문 다른 표현)에 cached 응답 반환해 LLM 호출 절감 (2) 경량 encoder(ModernBERT)로 복잡도 판단해 큰 모델/reasoning 필요 여부 결정 (3) task별 SLM(8~32B)로 적절 endpoint 라우팅 (4) agentic에서 MCP 기반 tool filtering·보안/정책 enforcement.
- semantic router 흐름: PII 마스킹(NER encoder + regex) → embedding(768차원) → vector search로 cached 응답/관련 tool 검색 → 소형 classifier로 LLM 선택·reasoning 여부 결정.

---

## 2. Performance Profiling Strategies
> LLM serving이 비싸 1%p 개선도 수백만 달러를 절약하므로 workload profiling이 운영 필수다.
- serving layer: throughput·TTFT·ITL·GPU/memory 활용 등 SLA/SLO 조정.
- framework layer: PyTorch Profiler로 operator(matmul·attention·layernorm) 실행·CPU-GPU 데이터 이동·compute/IO 중첩 검사 → dominant operator 교체, GPU idle 시 CPU 작업 async 이전.
- runtime layer: Nsight Systems(시스템 전역 timeline, GPU vs CPU/IO 병목 판별)·Nsight Compute(kernel 미세구조: warp occupancy·memory stall·tensor core).
- 전략: Nsight Systems로 시작 → GPU 저활용이면 PyTorch Profiler(CPU mode) → GPU busy인데 느리면 CUDA mode로 operator 특정 → 단일 kernel 지배면 Nsight Compute로 drill-down, 아니면 launch/overlap/sync 문제로 timeline 재검토.

---

## 3. Multimodal Serving — Input Processing
> text LLM을 VLM 등 multimodal로 확장 — 이 책은 multimodal 입력을 받되 표준 autoregressive decoder로 text를 생성하는 모델만 다룸(diffusion 기반 출력 생성은 제외).
- 입력: content에 image + text 전달 → prompt template에 <|vision_start|><|image_pad|><|vision_end|> 삽입, image placeholder ID가 644회 반복.
- 처리: text token은 text embedding으로, image placeholder는 Vision Encoder embedding으로 대체 — 이미지를 patch로 분할·hidden dim에 projection·patch 간 attention으로 공간/의미 관계 인코딩(dual-stream).

---

## 4. Multimodal — Architectural Implications
> multimodal 입력은 전처리 단계에 CPU-heavy 작업(고해상도 이미지 픽셀 변환·crop·resize·tensor 변환)을 도입해 early CPU bottleneck을 만든다.
- CPU가 큰 tensor 준비를 못 따라가면 빠른 GPU가 idle → throughput이 LLM이 아닌 vision 전처리 단계에 제약.
- vLLM V0→V1: CPU-intensive 작업을 GPU 실행과 분리한 완전 async 프로세스 채택. Process 0(API server·전처리·후처리)와 Process 1(GPU kernel scheduling/launch) 분리로, Process 0이 multimodal 전처리에 바빠도 Process 1의 kernel launch를 막지 않아 GPU 거의 full 활용.

---

## 5. Edge AI — Drivers & Enablers
> AI workload가 edge로 이동하는 동력: latency(robotics·자율주행·AR/VR의 ms 단위 반응), data locality(민감 데이터 로컬 처리·GDPR/HIPAA), cost(IoT·고해상 영상의 대역폭·전송 비용 절감).
- 단순 모델 축소가 아니라 hardware·compression·runtime·아키텍처 패턴의 융합:
- 저전력 하드웨어: NPU(수천 개 작은 처리 요소), TOPS/W(watt당 효율)가 edge 표준 지표.
- model compression: edge는 quantization·pruning·distillation이 기능의 전제 조건(on-device SRAM/flash 제약).
- heterogeneous compute: CPU(전처리)·NPU(compute-dense layer)·GPU(후처리) 분업 = HW 간 pipeline parallelism, subgraph partitioning.
- thermal-aware scheduling: fanless edge는 과열 시 throttle → 온도 모니터링으로 작은 모델 전환·core migration.
- edge-cloud hybrid: edge가 wake-word·전처리·tiny LLM, cloud가 대형 모델. bandwidth·battery·thermal·latency 기반 adaptive offloading.

---

## 6. Multi-LoRA Serving
> PEFT는 소수 파라미터만 적응하고 대부분을 freeze하며, LoRA는 저랭크 학습 layer를 삽입해 원본 weight 변경 없이 task별 적응을 빠르고 모듈식으로 학습한다.
- multi-LoRA serving: 여러 LoRA adapter를 GPU memory에 동시 적재해 한 instance에서 함께 서빙. 활성 adapter는 모두 적재(cold는 CPU/disk), continuous batching 유지 + Punica 같은 특화 kernel로 main weight + 다중 adapter 계산 결합.
- 이점: 단일 base 모델 instance가 다수 fine-tuned adapter를 동시 서빙해 GPU 수 절감(N개 GPU → 1개).
- 단, adapter별 트래픽이 많아 horizontal scaling이 필요하면 각 adapter를 main 모델에 merge해 독립 서빙이 나음 — multi-LoRA는 per-adapter 트래픽이 낮아 독립 서빙으로 HW 포화가 안 될 때 유용.

---

## 7. Model Serving in Reinforcement Learning
> RLHF는 labeled 데이터 학습을 넘어 human preference로 모델 응답을 정제해 더 helpful·safe·human-intent 정렬된 모델을 만든다(개방형 생성에 특히 유용).
- LLM serving in RL: OpenRLHF는 RLHF 학습 시간의 80%가 sample 생성(serving) 단계라 추정. vLLM·SGLang이 actor model(현재 policy)을 다중 replica로 배포해 대량 candidate 응답 생성 → reward model 점수화·reference model gradient 계산 → actor 업데이트·weight 동기화. serving 시스템이 RLHF 학습 루프의 필수 요소.
- determinism: replica/run 간 미세한 비결정성이 inconsistent reward·불안정 학습·재현 불가를 유발. batch-shape 변동 등이 수천 iteration에 누적 → 현대 RL serving 엔진은 batch-invariant deterministic inference로 재현 가능한 token 출력 보장.

---

## Summary (핵심 정리)
- semantic-aware caching/routing은 embedding·vector search로 의도 기반 cache hit·모델 선택·tool filtering·정책 enforcement를 endpoint 수준에서 수행한다.
- 성능 profiling은 serving→framework(PyTorch Profiler)→runtime(Nsight Systems/Compute)의 3계층 전략으로 병목을 root cause까지 추적한다.
- multimodal serving은 Vision Encoder dual-stream과 CPU 전처리 병목 완화(vLLM V1 async 프로세스 분리)를, edge AI는 NPU·compression·heterogeneous·thermal·hybrid를 요구한다.
- multi-LoRA는 다수 fine-tuned adapter를 한 instance에 공존시켜 GPU를 절약하고, LLM serving 엔진은 RLHF의 sample 생성을 떠받치며 deterministic inference가 throughput만큼 중요하다.
