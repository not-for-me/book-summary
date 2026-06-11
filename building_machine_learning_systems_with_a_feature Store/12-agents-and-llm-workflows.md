# 12. Agents and LLM Workflows

## 챕터 개요 (3줄 요약)
- LLM은 단순 wrapper에서 RAG·tool·memory·planning을 갖춘 agent로 진화했으며, context engineering이 핵심이다.
- prompt management·prompt engineering·RAG(vector/document/feature/graph)·function calling으로 prompt에 적절한 context를 채운다.
- MCP(tool)·A2A(agent) 프로토콜로 런타임 discovery를 표준화하고, workflow로 autonomy를 제약해 신뢰성을 높인다.

---

## 1. From LLMs to Agents
> chatbot → RAG → tool·자율성을 가진 agent로 진화.

- 초기 chatbot은 system prompt + query. cutoff 이후 사건 답변 불가 → **RAG**(query embedding으로 vector index에서 유사 chunk 검색 → prompt 추가 → in-context learning).
- vector embedding pipeline: chunking → embedding → vector index. RAG는 web search로도 확장.
- **agent**: tool 사용(MCP), LLM 호출(prompt+RAG context), trace 로깅, A2A로 capability 노출. application ID로 feature store에서 context 조회.

---

## 2. Prompt Management & Engineering
> task마다 system prompt + prompt template(system/user/assistant) 설계.

- chat template(**ChatML**: system/user/assistant). multimodal은 image 태그(Llama 4). LlamaIndex ChatPromptTemplate, Opik으로 버전 관리(source repo 또는 데이터 플랫폼).
- **prompt engineering 전략**: in-context learning(정적/RAG 예시), **CoT prompting**(step-by-step, regular LLM에서도 reasoning 유도, few-shot 동반), role-playing, structured output(JSON→function calling), prompt decomposition.
- **context window**: context length = 대화 + query + system prompt + 출력. 한계 시 warn/forget/summarize. input이 길수록 품질↓·지연↑(self-attention O(n²), flash attention/MoE로 완화).

---

## 3. Agents & Workflows with LlamaIndex
> stateful LLM workflow·agent 프레임워크. query engine, retriever, tool, memory.

- 추상화: query engine, retriever, tool(Python callable+metadata), settings, prompt template, memory.
- **workflow**: 개발자 정의 파이프라인(retrieve→context→summarize). 예측가능·신뢰성. LLM이 모든 next-step 결정하면 **agent**.
- **agent**: LLM + system prompt + tools. query → tool 호출 시퀀스 또는 응답 반환. tool 실행 → 결과를 대화 history에 추가 → 재호출 loop. 단계마다 신뢰성 없으면 error compounding → 복잡 작업은 deterministic workflow 선호.
- 예: `@step` 메서드로 FraudExplanationWorkflow(StartEvent→FetchExplanation→FindSimilar→StopEvent), FastAPI 노출, startup에 1회 초기화.

---

## 4. Retrieval-Augmented Generation
> 관련 context를 prompt에 주입. vector/document/feature/graph store.

- 큰 context window(1M token)라도 무관 정보는 품질↓ → 작고 관련된 context 유지. "Goldilocks" 예시 수.
- **chunking**: sentence/paragraph/fixed-token/semantic/recursive/sliding window. **lost context** 문제(chunk 독립 처리로 entity 참조 모호; 예: "Stockholm population" chunk에 "Stockholm" 없음). late-chunking 연구.
- LlamaIndex가 vector DB 추상화(VectorIndexRetriever + RetrieverQueryEngine). **reranking**(검색 후 relevance 재정렬, fine-tuned transformer).
- **document store**(OpenSearch/Elasticsearch, inverted index, **BM25**): 삽입 처리량↑·검색 지연↓(embedding보다 저렴). term frequency + IDF + 길이 정규화.
- **feature store RAG**: entity ID(deployment API에 포함)로 row 조회 → stringify → prompt. text2SQL은 아직 인간(92%)>LLM(77%), API/function calling이 더 잘 작동.
- **graph database RAG(GraphRAG)**: knowledge graph(node/edge)에 GQL/Cypher query. Text2Cypher도 probabilistic, templated query를 tool로 노출 권장.

---

## 5. Tools and Function-Calling LLMs
> query + 후보 함수 → LLM이 함수명·파라미터 JSON 반환 → 실행.

- function-calling LLM = JSON 출력 LLM. LlamaIndex FunctionTool + FunctionCallingAgent로 매핑 자동화.
- 흐름: query → function-calling LLM이 JSON 반환 → 함수 실행 → 결과를 2번째 LLM prompt에 context로 → 답변.
- 개선: 상세 system prompt(예시), 함수·파라미터 문서화, 작은 composable 함수로 refactor, 강력한 LLM.

---

## 6. Model Context Protocol (MCP)
> agent↔external tool/service 통신 표준(Anthropic 2024). N 프로토콜→1 프로토콜.

- JSON-RPC 2.0 transport, tool당 단일 input schema, deterministic·side-effect-free.
- building block: **primitives**(tools/resources/prompts), **discovery**(tools-list 등). 표준 JSON-RPC 에러.
- FastMCP: `@mcp.tool()`, `@mcp.resource()`, `@mcp.prompt()`. client는 URL 연결 후 `call_tool`. JSON/XML schema(XML 선호: validation·token 효율).
- 3 phase: initialization(discovery+버전 합의), usage(tool/resource 호출, elicitation/notification), termination.

---

## 7. Agent-to-Agent (A2A) Protocol
> agent↔agent discovery·협업 표준(Google 2025). JSON-RPC over HTTP/SSE.

- **Agent Card**(`/.well-known/agent.json`): identity, endpoint, auth, capabilities/tasks, input/output 형식.
- **task**: client가 요청하는 작업 단위(stateful·async). A2ACardResolver→A2AClient→send_message.
- MCP(intra-agent tool, JSON schema/contract, sync/async) vs A2A(inter-agent, 자연어, async 핵심) — 상호보완.

---

## 8. From LLM Workflows to Agents
> 제어 수준과 task 고정/런타임 discovery로 구분.

- workflow 패턴: prompt chaining(linear, CoT), parallelized orchestration(병렬, Anthropic multi-agent research), **routing**(router LLM이 task 선택, 코딩 agent).
- **agentic workflow**(=agent): tool/agent discovery(MCP/A2A) → planning(subtask 분해·순서) → execution → reflection. 동적 control flow, JSON 출력.
- 예: 카드 fraud 고객지원 agent("왜 fraud로 표시?") = feature store에서 transaction 조회 → interpretable feature를 LLM에 설명 요청.

### Planning
- LLM은 planning 약함(LeCun: autoregressive는 고정 계산량). → **LRM**(reasoning, `<think>` 토큰, System 2). novel plan vs 암기 논쟁.
- router LLM(classifier)이 가장 단순한 planner. 일반 planning은 subgoal·reward·time horizon. client로 plan 검증(specification), heuristic(invalid action 제거). planning·execution 분리(디버깅·trace).
- 권장: workflow부터 시작, 필요 시에만 agent(미정의 문제 + MCP/A2A 서비스 가용 시).

### Security
- 자율 agent는 control 획득 subgoal 위험(Hinton). untrusted input + private 정보 접근 = 악몽(instruction injection). 입력 제약·guardrail(ch14). 신뢰된 라이브러리만(supply chain).
- **domain-specific intermediate representation**: 사용자가 도메인 언어로 피드백(Lovable 웹페이지, Brewer YAML pipeline). 좋은 prompt는 재사용 자산.

---

## 9. Development Process & Hopsworks Deployment
> 모든 단계(query/MCP/RAG/LLM) 로깅 → error analysis → eval로 개선 측정.

- vibe coding 대신 엄격한 방법론(한 단계 변화가 품질 급락). trace 로깅 → error analysis(ch14) → eval(direct grading 1-5/Pass-Fail, LLM-as-judge, benevolent dictator).
- Hopsworks: agent를 Knative 컨테이너로, A2A API + MCP + RAG(feature store/vector index) + Opik trace + vLLM/KServe. `@hopsworks.a2a.agent()`/`@hopsworks.a2a.skill()`. Envoy AI Gateway(LLM 교체·rate limit·cost·metrics·보안), A/B testing.

---

## Summary (핵심 정리)
- LLM workflow·agent는 system prompt와 RAG로 prompt에 적절한 정보를 채워 task를 푸는 다양한 자율성의 프로그램이다.
- workflow로 autonomy를 제약하면 더 신뢰성 있는 LLM 서비스를 만든다.
- 추세는 tool·agent를 discover·사용하는 자율 agent로 향하며, MCP·A2A 표준이 중요하나 초기 단계다.
- security·planning 과제가 남아 있다.
