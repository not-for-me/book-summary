# 12. Designing LLM-Powered Systems

## 챕터 개요 (3줄 요약)

- LLM은 기존 MLOps infra를 대체하지 않고 확장한다 — DakkaBot(RAG) case study로 vector DB·prompt 관리·guardrail 등 LLM-specific 요소를 추가한다.
- 문서 ingestion → embedding → FAISS 인덱스 → LangChain retrieval → Gemini 생성으로 RAG를 구축하고 Chainlit UI를 붙인다.
- LLM 고유의 nondeterminism·multistep reasoning은 Langfuse observability로 tracing·cost·latency를 추적해야 한다.

---

## 12.1 LLMOps: New challenges, familiar principles

> LLM은 ML app 사고방식을 바꾸지만 K8s·monitoring·CI/CD 기반은 그대로 필수다.

### LLM app의 차이점

- **Nondeterministic output**: 같은 prompt도 sampling·temperature로 다른 응답 → `assert == expected` 무의미, semantic evaluation 필요.
  - **sampling param**: **temperature**(0=결정적/most probable, 0.7~1.0=창의적), **top-K**(K개 후보로 제한), **top-P**(누적 확률 P%). factual retrieval은 temp 0, 창의/대화는 controlled randomness.
- **Token limit/context window**: input(GPT-4 Turbo 128K, Claude 3.5 200K) + output(max_tokens). RAG는 chunk 수·길이·max_tokens가 architectural 결정. 초과 시 실패/truncate.
- **Multistep reasoning chain**: 선형 data-in/prediction-out이 아닌 retrieve→synthesize→generate→validate → debugging 어려움.
- **Prompt engineering**: feature engineering급 중요. prompt = versioning·testing 필요한 code(자연어).
- **Token-based cost**: uptime이 아닌 token 수 기반, 복잡한 query가 100x 비쌀 수 있음.

### ML platform 확장

- **유지**: infra(container·load balancer·auto-scaling), CI/CD, MLflow(prompt·param·eval tracking), monitoring(Prometheus/Grafana), security(단 prompt injection·jailbreak·data poisoning 등 신규 공격 벡터).
- **신규 metadata**: prompt artifact(system/template/few-shot), config(token·sampling·model), retrieval(vector index·chunking·threshold), eval artifact. 전체 artifact ecosystem versioning 필요.
- **신규 component**: **vector DB**(고차원 embedding, semantic search), **prompt management**(Git·A/B·rollback), **LLM gateway**(provider 라우팅·semantic caching·token rate limit), **specialized monitoring**(token·cost·trace·safety).

### 필수 tool

- **embeddings**: text→고차원 vector(의미 포착). keyword가 아닌 의미 기반 검색.
- **FAISS**(Facebook AI Similarity Search): 벡터 유사도 검색 엔진. dev·중규모용(대규모는 Pinecone/Weaviate/Chroma).
- **LangChain**: LLM app framework. document loader/text splitter/embedding/vector store/LLM/chain 표준 인터페이스. component 교체 용이. 복잡 워크플로는 **LangGraph**(graph 기반, 조건 분기·병렬·반복).
- **Langfuse**: open source LLM engineering platform. prompt tracing·token·cost·time-to-first-token, prompt 관리.
- **Promptfoo**: open source LLM security(취약점·jailbreak 탐지, 13장).

---

## 12.2 Building DakkaBot: A simple RAG architecture

> DataKrypt의 내부 개발자 assistant. 문서 ingestion→vector DB→LLM+embedding→orchestration→UI.

### 왜 RAG인가

- 단일 LLM call 한계: context window 제약, reasoning 저하, error propagation, cost 비효율.
- RAG 해결: training cutoff 이후 정보 접근, 선택적 retrieval(noise↓), real-time data, private data, cost 효율, **hallucination 완화**(source attribution·grounding).

### Google Gemini

- `text-embedding-004`(embedding) + `gemini-2.5-flash`(생성, 속도·cost 최적). free tier 실험용. exponential backoff·fallback(OpenAI/Claude) 권장.

### RAG 3 component

- **Retrieval**: `GeminiEmbeddings` wrapper — **document(RETRIEVAL_DOCUMENT)와 query(RETRIEVAL_QUERY) 비대칭 embedding**(정확도 10~20%↑). 품질 요인: chunking, embedding model, similarity metric(cosine), retrieval param(k). 인덱싱 pipeline: load(DirectoryLoader) → **chunk**(RecursiveCharacterTextSplitter, size 1000/overlap 200, 자연 경계) → FAISS index → persist(save_local/load_local, allow_dangerous_deserialization 보안 주의).
- **Augmentation**: retrieved chunk + query를 prompt로 조립(concatenation). persistent index 로드 시 동일 embedding model 필수.
- **Generation**: `ChatGoogleGenerativeAI(temperature=0.)`로 일관성. context+question+answer 구조 prompt로 hallucination 완화·grounding.

```python
# 전체 RAG (핵심 흐름)
docs = retriever.invoke(query)
context = "\n\n".join([d.page_content for d in docs])
prompt = f"Context: {context}\nQuestion: {query}\nAnswer:"
response = llm.invoke(prompt)
```

```text
Build-time: docs -> load -> chunk -> embed -> [FAISS index]
Query-time: query -> embed -> retrieve -> augment(context) -> LLM -> answer
```

---

## 12.3 Giving DakkaBot a UI (Chainlit)

> Chainlit = 대화형 AI 인터페이스 Python framework(streaming·history·source 표시). frontend 개발 불필요.

- **DakkaBotCore** 클래스로 리팩터(separation of concerns): async `initialize`(lazy), `process_query`(structured dict: response/sources/doc_count/error), `process_query_stream`(AsyncGenerator, token-by-token). singleton 패턴으로 재초기화 방지.
- **system prompt** = LLM의 "personality chip": output 표준화, 오해 방지, data leak 방지(instruction 노출 금지), persona, guardrail.
- Chainlit: `@cl.on_chat_start`(초기화·onboarding), `@cl.on_message`(input validation → thinking indicator → streaming token). author attribution으로 message 구분.

---

## 12.4 Observability for LLM applications

> 전통 관측성(CPU/memory/latency)은 "언제 고장났나"만 알려줄 뿐 "왜 나쁜 답인가"는 못 한다.

- reasoning chain tracking 필요: 어떤 문서 retrieve, context 조립, model 출력 이유, 실패 지점.
- **Langfuse 설정**: Docker(`docker compose up`, port 3000) → admin 계정·project·API key.
- **통합**: callback handler(`CallbackHandler()`)를 `config={"callbacks":[...], "metadata":{...}}`에 추가 → LLM call·retrieval·chain·error·cost 자동 tracking. "set once, trace everywhere".
- **dashboard**: traces timeline(사용 패턴), Model Usage cost(gemini-2.5-flash $0.62/수백 query — model 선택이 cost 좌우), token consumption, latency percentile(trace/generation/span). trace 상세: latency·input/output token·cost·전체 conversation flow.

### 전통 metric의 한계

- accuracy: binary right/wrong은 partial correctness·intent·consistency·hallucination·subjective task 못 잡음.
- latency: time-to-first-token vs total, 체감 반응성, resource 패턴 무시.
- 필요: multistep reasoning trace(reasoning path·error propagation·token allocation·branching), token consumption monitoring(input/output ratio, context utilization, query type별 cost, waste 탐지).

---

## Summary (핵심 정리)

- RAG의 힘은 분리 — retrieval과 generation을 decouple해 model 재학습 없이 knowledge 갱신.
- temperature=0은 결정적이 아닌 "일관적" — traditional assertion 대신 semantic evaluation.
- token budget은 architectural 제약(50 chunk × 500 = 25000 token) — 초과 시 실패/truncate.
- LLM call 전 input sanitization으로 security·cost 동시 확보(prompt injection 조기 차단).
- 전통 metric은 "언제 고장" only, tracing(Langfuse)이 "왜 나쁜 답"을 설명.
- "model"은 이제 pipeline — prompt·embedding·retrieval·config를 독립 versioning, prompt를 code처럼.
- LangChain으로 시작 → LangGraph로 졸업(조건 분기·병렬·반복). 다음 챕터(13장)에서 production LLM system(testing·safety·governance)을 다룬다.