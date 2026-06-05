# 10. AI Engineering Architecture and User Feedback

## 챕터 개요 (3줄 요약)
- 가장 단순한 foundation model 애플리케이션 구조에서 출발해, 문제를 해결하며 컴포넌트(context, guardrails, router/gateway, cache, agent pattern)를 점진적으로 추가하는 production 아키텍처를 제시한다.
- 복잡해진 시스템을 운영하기 위한 monitoring/observability(metrics, logs, traces, drift detection)와 orchestration의 역할을 설명한다.
- 대화형 인터페이스에서 얻는 user feedback이 평가·개발·개인화 및 data flywheel의 핵심 데이터 자원임을 강조하고, 수집 설계와 한계(bias)를 다룬다.

---

## 1. AI Engineering Architecture

> 단순 구조(query→model→response)에서 시작해 필요에 따라 컴포넌트를 단계적으로 추가하는 일반화된 production 아키텍처.

가장 단순한 구조는 context 보강, guardrails, 최적화가 전혀 없는 형태다. 여기서 출발해 요구가 생길 때마다 컴포넌트를 추가한다. Model API box는 third-party API와 self-hosted 모델 모두를 의미한다. 모든 사람의 요구가 다르므로 자신의 애플리케이션에 맞는 순서로 확장하면 된다.

```
        +-------+      +-----------+      +----------+
 User-->| Query |----->| Model API |----->| Response |--> User
        +-------+      +-----------+      +----------+
```

### Step 1. Enhance Context
> 모델이 각 query에 답하는 데 필요한 관련 context를 구성하는 메커니즘을 추가한다.

Context 구성은 foundation model의 feature engineering과 같다. text/image/tabular retrieval과 web search 같은 tool로 정보를 자동 수집한다. 출력 품질에 핵심적이라 대부분의 모델 API가 지원하지만, 업로드 가능한 문서 수, retrieval 알고리즘, chunk size, tool 종류·실행 방식 등에서 제공자마다 차이가 크다.

### Step 2. Put in Guardrails
> 위험에 노출되는 지점마다 input/output guardrail을 두어 시스템과 사용자를 보호한다.

Input guardrail은 외부 API로의 민감정보 유출과 악성 prompt를 막는다. 민감정보(PII)는 차단하거나 [PHONE NUMBER]처럼 masking 후 reverse map으로 unmask한다. Output guardrail은 품질 실패(잘못된 형식, hallucination)와 보안 실패(toxic, 사적정보, brand-risk)를 잡고 정책을 정한다. retry(병렬 호출 포함), human fallback을 활용하되 latency·cost trade-off를 고려한다. stream mode에서는 부분 응답 평가가 어려운 한계가 있다.

### Step 3. Add Model Router and Gateway
> 여러 모델을 다룰 때 router는 query를 적절한 모델로 보내고, gateway는 통합·보안 인터페이스를 제공한다.

Router는 intent classifier로 query 의도를 예측해 전문화 모델/저렴한 모델/human/FAQ로 분기하고, out-of-scope·모호한 query 처리, next-action 예측에 쓰인다. 작고 빠르게 만들어야 한다. Gateway는 모델별 통합 wrapper로 코드 유지보수를 쉽게 하고, access control·cost 관리·fallback·logging·caching을 중앙화한다. Portkey, MLflow Gateway, Kong, Cloudflare 등 off-the-shelf 솔루션이 많다.

### Step 4. Reduce Latency with Caches
> exact cache와 semantic cache로 latency와 cost를 줄인다.

Exact cache는 동일 요청에만 캐시를 사용하며 LRU(Least Recently Used)·LFU(Least Frequently Used)·FIFO 등 eviction policy가 필요하다. 사용자별·시간민감 query는 캐시하지 않는다. Semantic cache는 의미적으로 유사한 query에 재사용하며 embedding+vector search+유사도 threshold에 의존해 실패 지점이 많고 잘못된 응답 위험이 있다. 캐시 처리 부주의는 사용자 간 data leak를 유발할 수 있다.

### Step 5. Add Agent Patterns
> loop, 병렬 실행, conditional branching 같은 agentic pattern으로 복잡한 애플리케이션을 만든다.

생성된 출력을 다시 retrieval과 결합해 모델에 재입력하는 loop를 구성할 수 있다. write action(이메일 작성, 주문, 송금)은 시스템 능력을 크게 키우지만 위험도 크게 높이므로 극도로 신중해야 한다. 복잡성이 커질수록 failure mode도 늘어 디버깅이 어려워진다.

```
 Query -> Context -> Guardrail -> Router/Gateway -> Model -> Cache
                                                      |
                              (Agent loop: feedback) <+
```

---

## 2. Monitoring and Observability
> observability는 사후 보강이 아니라 제품 설계에 통합되어야 하며, 시스템이 복잡할수록 더 중요하다.

목표는 평가와 같다: 위험 완화와 기회 발견. DevOps 지표로 MTTD(Mean Time To Detection), MTTR(Mean Time To Response), CFR(Change Failure Rate)을 활용한다.

### Metrics
> 지표 자체가 목적이 아니라, 무엇이 잘못됐는지 알리고 개선 기회를 찾기 위한 수단이다.

먼저 잡으려는 failure mode를 정의하고 그에 맞춰 지표를 설계한다. format 실패(잘못된 JSON), factual consistency, toxicity·PII, guardrail 발동률, refusal rate를 추적한다. 대화 신호(조기 중단율, 턴 수, 입력/출력 token 길이·분포)도 품질 추론에 쓰인다. latency 지표 TTFT(Time To First Token), TPOT(Time Per Output Token), total latency와 cost(TPS, Tokens Per Second)를 사용자·릴리스·버전별로 분해한다. north star(DAU 등)와의 상관도 분석이 유용하다.

### Logs and Traces
> metric은 무언가 잘못됐음을 알리고, log/trace는 무엇이 일어났는지 알려준다.

"모든 것을 log하라"가 원칙이다: 설정(모델 endpoint, sampling 설정, prompt template), user query, 최종 prompt, 출력, 중간 출력, tool 호출·결과, 컴포넌트 시작/종료/crash. tag와 ID를 붙인다. Trace는 관련 이벤트를 연결해 요청의 전체 실행 경로를 재구성하며, 실패 시 정확히 어느 단계가 잘못됐는지 짚어준다.

### Drift Detection
> system prompt, user behavior, 기반 model의 변화가 성능에 영향을 준다.

prompt template 수정으로 system prompt가 모르는 사이 바뀔 수 있다. 사용자는 더 좋은 결과를 얻으려 행동을 바꾼다(응답 길이 점진 감소 등). API는 그대로지만 기반 model이 갱신될 수 있어(예: GPT-4의 버전 간 벤치마크 차이) 직접 변화를 감지해야 한다.

---

## 3. AI Pipeline Orchestration
> orchestrator는 여러 컴포넌트가 end-to-end pipeline으로 협력하도록 정의·연결한다.

두 단계로 동작한다: (1) components definition — 사용하는 모델·데이터소스·tool 정의, (2) chaining — query 수신부터 완료까지의 단계를 함수 합성처럼 연결. 컴포넌트 간 데이터 형식을 보장하고 흐름이 깨지면 알려야 한다. RAG(Retrieval-Augmented Generation)/agent 프레임워크가 곧 orchestration 도구인 경우가 많다(LangChain, LlamaIndex, Haystack 등). 처음엔 없이 시작하고, 후반부에 통합·확장성·복잡 pipeline 지원·사용성을 기준으로 도입을 평가한다. (일반 workflow orchestrator인 Airflow와는 다르다.)

---

## 4. User Feedback
> AI 애플리케이션에서 user feedback은 평가·개발뿐 아니라 모델 개선용 proprietary 데이터로서 경쟁 우위(data flywheel)를 만든다.

user feedback은 user data이므로 프라이버시 존중과 사용 목적 고지가 필요하다.

### Extracting Conversational Feedback
> 명시적(explicit) feedback과 행동에서 추론하는 암묵적(implicit) feedback이 있으며, 대화형 인터페이스는 새로운 implicit 신호를 만든다.

Explicit feedback(thumbs up/down, 별점)은 해석이 쉽지만 sparse하고 response bias가 있다. Natural language feedback 신호로는 early termination(조기 중단), error correction("No,…"/"I meant,…"/직접 편집), complaints, sentiment, 모델의 refusal rate가 있다. 사용자 편집은 (query, winning, losing) preference 데이터가 된다. 기타 신호로 regeneration, 대화 organization(delete/rename/share), conversation length, dialogue diversity가 있다. implicit feedback은 풍부하지만 noisy해 사용자 연구가 필요하다.

### Feedback Design
> feedback은 사용자 흐름에 비침투적으로 통합되어야 하며, 언제·어떻게 수집할지가 핵심이다.

수집 시점: 초기 calibration(대개 선택적), 문제 발생 시(downvote, regenerate, human 전환), 모델 확신이 낮을 때(side-by-side 비교로 preference 수집). 방법: 추가 노력 없이 워크플로에 녹이기. Midjourney(upscale/variation/regenerate), GitHub Copilot(Tab 수락/계속 입력 거절)이 좋은 예다. 통합형 제품(Gmail, Copilot)이 standalone(ChatGPT)보다 고품질 feedback 수집에 유리하다. 깊은 분석엔 직전 대화 맥락이 필요한데 PII 문제로 사용자 동의가 필요하다. 신호의 공개/비공개 여부는 사용자 행동과 신호 품질에 큰 영향을 준다.

### Feedback Limitations
> feedback에는 bias가 있고, 무분별하게 쓰면 제품을 망칠 수 있다.

주요 bias: leniency bias(과도하게 후한 평가), randomness(무성의한 응답), position bias(앞 옵션 선호), preference bias(긴 응답·recency 선호). Degenerate feedback loop는 예측이 feedback에 영향을 주고 그것이 다시 모델에 반영되며 초기 bias를 증폭한다(exposure/popularity bias, filter bubble). user feedback으로 학습하면 모델이 사실보다 사용자가 원하는 답을 주는 sycophancy로 흐를 수 있다.

```
 Prediction -> shown to user -> Feedback -> next model -> Prediction
        ^----------------- amplifies bias -----------------+
```

---

## Summary (핵심 정리)
- 단순 구조에서 시작해 context, guardrails, router/gateway, cache, agent pattern을 점진적으로 더하는 일반 아키텍처를 제시했고, 컴포넌트 경계는 유동적이다(예: guardrail은 여러 위치에 구현 가능).
- 컴포넌트 추가는 능력·안전·속도를 높이지만 복잡성과 새로운 failure mode를 만들어, monitoring/observability(metric·log·trace·drift)가 필수다.
- 대화형 인터페이스는 새로운 user feedback을 가능케 하며, 이는 분석·제품 개선·data flywheel의 핵심으로 AI engineering이 product에 더 가까워지게 만든다. feedback의 bias와 한계를 이해하고 설계해야 한다.
