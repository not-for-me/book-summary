# 03. Model Serving System Design: A Deep Dive

## 챕터 개요 (3줄 요약)
- single-model과 multi-model serving 시스템을 first-principles로 직접 구현하며 batching·streaming·routing·isolation·resource management의 조립 방식을 보여준다.
- single-model serving의 핵심 컴포넌트(API server·LLM engine·workload manager·model executor·model worker·model manager)와 multi-process(CPU 오케스트레이션 / GPU 격리) 패턴을 다룬다.
- multi-model serving은 cost-optimized(자원 공유·on-demand 로딩)와 latency-optimized(모델별 dedicated 그룹·사전 provisioning) 두 설계의 trade-off로 정리된다.

---

## 1. Build an Online LLM Serving Service from Scratch — Design Goals & Architecture
> 단일 LLM을 로드해 batch·streaming 동시 요청을 처리하는 간소화된 serving service를 직접 만들어 framework가 자동화하는 request handling·batching·streaming·scheduling·resource management를 노출한다.
- 6개 핵심 컴포넌트: API server(HTTP), LLM engine(오케스트레이터), workload manager(queue·batch 관리), model executor(worker 조율·cross-process 호출), model worker(전용 process에서 inference), model manager(모델 로드·캐시).
- multi-process 패턴 이유: GPU는 비싸므로 idle을 막기 위해 model 실행을 전용 process(GPU 바인딩)에 격리하고, web service(CPU)는 동시 요청 관리·orchestration에 집중 → GPU 활용 극대화.

---

## 2. Single Generation Request Handling
> LLMEngine이 ModelExecutor·WorkloadManager를 초기화하고, ModelExecutor가 task_queue/result_queue로 별도 process의 ModelWorker와 통신해 한 번에 prompt 하나를 처리한다.
- ModelWorker는 while loop로 task_queue 모니터링 → prompt pull → inference → result_queue에 결과 push.
- 한계: 한 번에 prompt 하나만 처리 → 낮은 throughput, compute 자원 미활용 → batching 필요.

---

## 3. Batching
> 여러 request의 prompt를 묶어 큰 batch로 LLM에 실행하고(자원 활용 극대화), 생성 결과를 prompt ID로 원 request에 정확히 매핑한다.
- 각 prompt를 Sequence 객체(고유 ID)로 추적, WorkloadManager가 in-memory로 관리하며 다음 batch 결정.
- 워크플로: request intake → prompt queuing → 추적·batching(FIFO) → batch 실행 → inference → ID 기반 response mapping.
- batch size·전략은 throughput에 큰 영향 — 모델·prompt 특성·traffic·HW에 따라 튜닝 필요(dynamic/continuous batching은 ch6~7). prompt별 개별 추적은 web request와 model 실행을 분리해 최적화 유연성 제공.

---

## 4. Streaming with Batching
> 내부적으로는 매 generation step마다 batch로 처리하되, 생성된 token을 즉시 사용자에게 stream하여 high throughput과 실시간 응답성을 동시에 달성한다.
- 변경점: 생성 API를 async로, ModelWorker가 step당 token 하나 생성, WorkloadManager가 부분 출력 추적, prompt마다 event queue 보유, LLMEngine에 전용 batch-processing thread.
- 구현: background thread(requests_processing_loop)가 batch를 돌리며 token을 각 prompt의 event queue로 dispatch → API server가 SSE(Server-Sent Events, text/event-stream)로 client에 전달.
- 완료된 prompt는 batch에서 제거해 신규 요청 공간 확보 (예: T2 완료된 Prompt1을 T3에 제거).

---

## 5. Batch Serving with vLLM
> production-grade framework인 vLLM은 model 로드·memory 관리·dynamic batching·streaming·scheduling을 고도로 최적화해 약 10줄로 batch inference를 구현하게 해준다.
- LLMEngine이 VLLM(model=...)을 초기화하고 generate 요청을 그대로 위임 → 앞선 streaming 설계보다 아키텍처가 훨씬 단순.
- 단, default 값으로 충분치 않으면 customizing 필요 — 내부 동작 이해가 max_batch_size·max_num_seqs 등 튜닝과 ch6~7 최적화 옵션 활용의 기반.
- 배포 방식 2가지: library 임베드(긴밀 통합·제어) vs standalone web server(REST/streaming API 노출). 실무는 후자가 흔함.

---

## 6. A General Design for Single-Model LLM Serving
> production serving은 low latency·high throughput(QPS/TPS)·scalability·reliability/availability·resource efficiency·observability를 충족해야 하고, LLM은 추가로 큰 model size·KV cache 관리·streaming·가변 길이 workload batching 과제를 가진다.
- 핵심 설계 원칙: 요구사항을 3영역으로 분리 — service infrastructure management(scaling·availability·monitoring), business logic handling(통합·batching·streaming), model serving performance(latency·throughput·LLM 최적화).
- part A: serving 로직을 container/Pod로 캡슐화 + 분산 compute system(K8s·cloud)에 인프라 책임 위임, load balancer로 트래픽 분배.
- part B(serving frontend): 인증·인가, 외부 시스템 통합, 모델 다운로드·config, 요청 전처리·batching, rate limiting.
- part C(serving backend): 별도 process로 실제 inference, vLLM·Triton 등으로 quantization·KV cache·continuous batching·GPU 활용 최적화. 진화하는 요소를 안정적 인프라에서 격리하는 것이 핵심.

---

## 7. Build a Multi-Model Serving Service from Scratch
> 다양한 크기·버전·task별 모델을 공유 인프라에서 동적으로 관리·라우팅하여 hardware 활용을 높이고 운영 overhead를 낮춘다.
- 5개 컴포넌트: API server, model manager(캐시·worker 라이프사이클), model store(metadata), model engine(metadata 기반 worker 생성), model worker(로드·inference).
- 워크플로: client가 model_id+input 전송 → cache 조회 → 미적재 시 metadata fetch → ModelWorker 생성 → cache 등록(가득 차면 LRU eviction) → inference → 응답.
- 3대 설계 요구: cross-framework 지원(transformers/torchvision worker 분리), unified API(generic predict 인터페이스, 전·후처리는 client 책임), resource management(lazy loading + LRU eviction).

---

## 8. Using NVIDIA Triton as a Model Server
> Triton은 PyTorch·TensorFlow·ONNX·TensorRT 등 다양한 포맷을 일관된 HTTP/gRPC API로 serving하는 대표 multi-model 솔루션으로, model management API와 inference API를 노출한다.
- 사용 흐름: 모델을 repository에 복사 → management API로 load → inference API로 예측.
- 통합: TritonWorker(wrapper)가 _load_model로 Triton에 모델 적재, predict로 입력을 Triton 포맷 변환 후 client.infer() 호출, __del__에서 unload로 자원 회수.
- 패턴: web 요청·시스템 통합·자원/파일 관리는 wrapper service가, 핵심 inference는 Triton이 담당.

---

## 9. Trade-offs in Multi-Model Serving Designs — Challenges
> multi-model serving은 비용 절감에 유리하나 두 가지 UX 과제가 있다: cold start latency(미적재 모델 로드 지연, 고트래픽 시 timeout·연쇄 실패)와 hot model scaling(인스턴스별 독립 캐시로 인한 복제·라우팅 복잡성).
- 적합 사례: 하루 몇 시간만 도는 batch job, 1,000개 고객 모델 중 동시 200개만 호스팅 등 on-demand 로딩.

---

## 10. Cost-Optimized vs Latency-Optimized Multi-Model Design
> cost-optimized는 다수 모델이 자원을 공유하고 on-demand 로딩하며, model service API/routing이 적재된 인스턴스로 라우팅·replica 추적·bin-packing으로 자원 최소화한다(단 reactive해 급증 시 latency 불가피).
- latency-optimized는 모델마다 dedicated single-model instance group을 두고 model-provisioning service로 사전 적재 → cold start 없음, 독립 scaling, 단순한 유지보수. 단 미사용 모델 overprovisioning으로 비용 비효율.
- 정답은 없다 — 비용·성능·운영 단순성 목표에 따라 아키텍처를 맞춤화. LLM에도 prefix caching/routing, multi-LoRA 관리에 multi-model 패러다임이 유효(ch7·ch10).

---

## Summary (핵심 정리)
- single-model serving은 API server·workload manager·model executor·model worker 협업으로 동시 요청·batching·streaming을 처리하며, GPU 활용을 위해 model 실행을 전용 process로 격리한다.
- prompt별 Sequence 추적은 web request와 model 실행을 분리해 FIFO batching·token streaming(SSE)·동시 batching 같은 최적화를 가능케 한다.
- vLLM·Triton 같은 framework는 복잡성을 추상화하면서 튜닝 여지를 제공하므로, 내부 동작 이해가 효과적 설정과 아키텍처 결정의 핵심이다.
- multi-model serving은 LRU 기반 on-demand 로딩으로 cross-framework 모델을 공유 인프라에 호스팅하며, cost-optimized(자원 공유)와 latency-optimized(dedicated 그룹·사전 provisioning) 설계는 비용·성능·운영 복잡성의 trade-off를 보여준다.
