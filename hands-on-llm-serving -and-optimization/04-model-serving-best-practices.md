# 04. Model Serving Best Practices

## 챕터 개요 (3줄 요약)
- 현대 LLM 애플리케이션은 단일 request-response가 아니라 agentic workflow에 내장되어 multiple LLM call·retrieval·tool 실행을 일으키므로 serving이 시스템 아키텍처 문제로 확장된다.
- enterprise LLM serving은 public API부터 model layer까지 layered reference architecture로 구성되며, open source(Kubernetes·Ray Serve·vLLM) 또는 cloud vendor(AWS SageMaker)로 구현 가능하다.
- build vs cloud는 binary가 아닌 control 정도의 spectrum이며, 모든 설계 결정은 latency·throughput 같은 정량적 metric으로 평가해야 한다.

---

## 1. Model Serving in an Agentic World
> agent는 high-level goal을 이해하고 reasoning·tool 선택·중간 결과 기반 반복을 통해 최소 개입으로 결과를 산출하는 autonomous LLM 시스템으로, 단일 interaction이 다수의 LLM call을 유발해 serving 요구를 재편한다.
- 전통 rule-based assistant와의 근본 차이는 autonomy — 자연어 해석·계획·tool 사용·feedback 적응.
- agent는 token 사용량·tail latency를 증폭하고 dynamic compute·orchestration을 요구 → caching·batching·scheduling 최적화의 직접 동기.

---

## 2. A Sample Knowledge Agent & Workflow
> PDF를 질의·분석하는 Knowledge Agent는 orchestrator + RAG + Planner + Actions(executor)로 구성되고, OpenAI embedding(text-embedding-3-small)과 LLM(gpt-4.1-nano)으로 동작한다.
- 워크플로: 질문 → Planner가 LLM으로 실행 plan 생성(예: query_rag_with_context → generate_analysis → generate_summary) → ActionExecutor가 순차 실행 → 결과 통합 반환.
- 각 action은 specialized prompt template의 LLM call로 구현 — 복잡한 질의를 단계로 분해해 autonomous 응답 합성.

---

## 3. Agent Autonomy — Actions, Planning, Tool Calling
> agent는 재사용 가능한 action(요약·질의·분석)을 정의하고 Planner가 LLM으로 고수준 지시를 subtask로 분해해 최적 실행 경로를 자동 결정한다.
- action은 LLM prompt뿐 아니라 tool call(web search·API·DB), system operation(파일·workflow), reasoning step을 포함할 수 있다.
- MCP(Model Context Protocol): LLM-tool 상호작용을 표준화해 통합을 하드코딩 대신 일관된 인터페이스로 처리, ad-hoc prompt engineering의 취약성 완화.
- tool calling 4단계: LLM이 reasoning → 구조화 출력(JSON) 생성 → tool 실행 → 결과를 LLM에 반환·반복.

---

## 4. RAG vs CAG
> RAG는 query 시점에 외부 지식을 retrieve해 prompt context를 확장(정적 지식·hallucination·도메인 gap 완화), 답변 품질을 높인다.
- RAG 워크플로: index-building(offline; 파싱→~1,000 token chunking→embedding→vector DB) + query/retrieval(online; query embedding→cosine similarity 검색→관련 chunk를 LLM에 전달).
- chunk size는 trade-off — 작으면 precision↑ 맥락 손실 위험, 크면 맥락↑ precision 희석. context window 한계 때문에 chunking 필요.
- CAG(Cache-Augmented Generation): 확장된 context window를 활용해 지식을 LLM KV cache에 사전 적재 → retrieval latency·시스템 복잡성 제거, serving 효율 향상.
- 둘은 경쟁이 아니라 다른 layer 문제 해결 — 지식 신선도는 RAG, latency·throughput·cost는 CAG. agentic workflow에선 함께 쓰는 경우 흔함.

---

## 5. How Agents Use Model Serving
> agent는 LLM(reasoning), embedding(retrieval), vision/speech(multimodal), tool-specific 모델과 외부 tool을 model serving service(HTTP/gRPC)로 on-demand 호출하며, 실시간·interactive 특성상 high-performance·low-latency·cost-efficient serving이 성패를 좌우한다.

---

## 6. LLM Serving in Enterprise Systems — Layered Architecture
> OpenAI·Anthropic 등이 채택한 enterprise serving은 인증·pricing·resource·networking·observability 등을 위해 수백~수천 엔지니어가 협업하는 layered 아키텍처로 구성된다.
- Public API layer: networking·auth·pricing·rate limit·routing (과제: high concurrency, fair usage·monetization, low-latency global access, security).
- Resource management layer: CPU/GPU/memory 관리 (capacity planning, GPU utilization, customer prioritization).
- Model selection/orchestration layer: 요청별 모델 선택(accuracy·latency·cost 균형), speculative decoding·model family routing.
- Distributed serving layer: 대형 모델 분산 호스팅 + distributed caching(KV·prompt·semantic), cache-aware routing.
- Core inference layer: vLLM·Triton·TensorRT-LLM·SGLang + 최적화 kernel(FlashAttention·PagedAttention).
- Model optimization layer + Model layer: 최적화 기법 적용 + 학습된 모델 production 이관·버전 관리.

---

## 7. Building with an Open Source Stack
> Kubernetes를 기반으로 resource 관리·networking·routing·scaling·monitoring을 처리하고, FastAPI(public API)·Ray Serve/Triton(model hosting)·vLLM(분산 serving·최적화)으로 enterprise stack을 조립한다.
- Public API: FastAPI로 /v1/chat/completions + JWT/API key auth(require_auth), Kubernetes HPA autoscaling(CPU 70% 초과 시 3~15 인스턴스), Nginx/Envoy ingress로 rate limit.
- Model selection: 요청 token size 기반으로 직접 forward 또는 speculative decoding(작은 draft 모델 생성 + 큰 target 모델 일괄 검증), tenant·region·cost 기반 endpoint 선택.
- Ray Serve single-model: @serve.deployment(num_replicas, num_gpus)로 vLLM QwenVLLM 호스팅.
- Ray Serve multi-model: model multiplexing(@serve.multiplexed, get_multiplexed_model_id)으로 공유 replica에서 다수 모델 LRU 캐시·동적 로드. HTTP header ray_serve_multiplexed_model_id로 모델 지정.

---

## 8. Building with a Cloud Vendor (AWS SageMaker 6가지 옵션)
> SageMaker는 fully managed부터 fully custom까지 ease-of-use ↔ control trade-off의 6가지 serving 옵션을 제공한다.
- Option 1 — Bedrock(fully managed FM): API 호출만, pay-as-you-go, zero DevOps, 커스터마이즈 최소. 프로토타입·즉시 추론에 적합.
- Option 2 — SageMaker JumpStart(one-click 배포): 선별 모델을 자기 계정 인프라에 시간당 과금으로 배포, instance type 선택 가능. 잘 알려진 모델 신속 호스팅.
- Option 3 — Bring Your Own Model(DLC): 프레임워크별 prebuilt 컨테이너(PyTorch·TF·HF·DJL/LMI)로 코드 없이 자기 모델 서빙. 컨테이너 내부 제어는 제한.
- Option 4 — Bring Your Own Code(Script Mode): vendor 컨테이너 위에 model.py(initialize·handle)로 custom 전·후처리·로딩 구현. 프레임워크/CUDA 버전은 이미지에 고정.
- Option 5 — Bring Your Own Serving Image: 자체 Docker 컨테이너 제공(언어·프레임워크 자유), SageMaker는 HTTP 계약(/invocations·/ping)만 요구. 빌드·유지 복잡성 부담.
- Option 6 — Build Your Own Infrastructure: cloud-managed K8s(EKS) 위에 Triton·vLLM·KServe·Ray 등 자유 구성, traffic·autoscaling·security·cost까지 소유. 최대 제어·compliance·cutting-edge 최적화.

---

## 9. Comparing Options & Build-or-Buy Strategy
> serving 선택은 one-size-fits-all이 없으며 ease-of-use ↔ control trade-off에서 시작해 cost·performance로 평가하고, 프로젝트 성숙·트래픽 증가에 따라 단순 옵션 → custom 옵션으로 이동한다.
- build vs cloud는 binary가 아닌 spectrum — 대부분 팀은 vendor 위에 custom handler·autoscaling·routing·cost control을 surgical하게 추가하는 중간 지점.
- build 방식을 이해하면 vendor feature 해독, trade-off 정량화, 80% default 유지+20% 교체, 디버깅이 가능.
- 전략: SLO 충족·비용 적정·feature 속도 중요 시 vendor managed 유지 / 특수 batching·routing·격리 필요 시 hybridize / HW·runtime·compliance·deep tuning 필요 시 BYO / 볼륨 안정·복잡성 비효율 시 vendor 복귀.

---

## 10. Measuring Performance in LLM Serving
> 모든 아키텍처 결정은 측정 가능한 결과(latency·throughput)로 평가해야 하며, 이는 후속 최적화 기법의 기준선이 된다.
- Latency: E2E latency(요청~전체 응답 완료 + queuing·network·routing 오버헤드), TTFT(첫 token까지; prefill phase·응답성), ITL/TPOT(이후 token 생성; decode phase). E2E ≈ TTFT + ITL×(N-1).
- 용도별 우선순위: agentic workflow는 E2E, chatbot streaming은 TTFT, 긴 출력은 ITL 중요. 모든 metric을 동시에 개선하는 기법은 없음 → trade-off 필요.
- Throughput: RPS/RPM(범용이나 input/output 길이·동시 사용자에 민감해 비교 부적합), TPS(생성 token/초; LLM 표준이나 input 단축·batch 최적화로 인위적 inflation 가능).

---

## 11. Best Practices for Performance Measurement
> 의미 있는 성능 측정은 granularity·realism·fairness·지속 점검의 조합을 요구한다.
- latency vs throughput trade-off 인식(offline은 throughput, interactive는 latency 우선), use-case별 acceptable latency 목표 설정(1초 달성 후 0.5초는 과잉 최적화일 수 있음).
- E2E를 TTFT·ITL로 분해해 bottleneck 파악, 실제 traffic 패턴(input/output 길이 분포·버스트) 시뮬레이션, 실험 일관성(한 번에 knob 하나).
- HW 활용률(GPU/CPU/memory) 모니터링, metric 인위적 inflation 회피, production 지속 모니터링, 주기적 test suite(regression·scalability·A/B).

---

## Summary (핵심 정리)
- 현대 LLM serving은 agentic workflow(planner·actions·RAG/CAG·tool calling)에 내장되어 단순 inference를 넘어 orchestration·memory·시스템 조율 문제로 확장된다.
- enterprise serving은 public API~model layer의 layered 아키텍처로 각 layer가 독립 진화하며, Kubernetes·Ray Serve·vLLM 같은 open source 또는 SageMaker 6옵션으로 구현된다.
- build vs cloud는 control 정도의 spectrum이며, 대부분 팀은 vendor 위에 targeted customization을 더하고 트래픽·비용에 따라 위치를 동적으로 조정한다.
- 모든 설계는 latency(E2E/TTFT/ITL)와 throughput(RPS/TPS)으로 평가하며, 공정한 벤치마크·현실적 traffic 시뮬레이션·지속 모니터링이 후속 최적화의 토대다.
