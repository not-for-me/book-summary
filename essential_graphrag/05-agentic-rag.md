# 05. Agentic RAG

## 챕터 개요 (3줄 요약)
- agentic RAG는 여러 retrieval agent 중 질문에 가장 적합한 것을 선택·실행하는 시스템이다.
- 핵심 3요소: retriever router(최적 retriever 선택), retriever agents(실제 검색), answer critic(답변 검증).
- LLM의 tool/function-calling 능력으로 routing·검증을 구현하며, 일반·특화 retriever를 점진적으로 추가한다.

---

## 1. What is agentic RAG?
> 시스템이 사용자를 대신해 어떤 retriever를 쓸지 선택하고, 찾은 context가 질문에 답하는지 판단한다.

- **Retriever router**: 질문(들)을 받아 최적 retriever(들)를 반환하는 함수(보통 LLM).
- **Retriever agents**: 실제 검색기. 광범위(vector similarity)부터 특화(파라미터 받는 hardcoded Cypher)까지.
  - generic: vector search(unstructured), text2cypher(structured). production에서 기대치 충족은 어려움.
  - specialized: 좁지만 특정 질문에 매우 정확. 시간이 지나며 추가 구축.
- **Answer critic**: retriever 답변이 원 질문에 옳게 답하는지 검증. 불완전 시 차단 후 새 질문 생성해 재시도. 무한루프 방지 위해 exit criteria 필요.

---

## 2. Why do we need agentic RAG?
> 다양한 데이터 소스 중 최적을 쓰거나, 복잡한 소스에 특화 retriever가 필요할 때 유용하다.

- 데이터가 복잡하면 text2cypher가 올바른 query 생성에 실패 → 특화 retriever(좁은 text2cypher나 파라미터 hardcoded query)로 보완.
- text2cypher를 catchall로 두고, 특정 질문은 전용 retriever로 처리.
- production에서 성능·답변 일관성 유지에 효과적.

---

## 3. How to implement agentic RAG
> retriever tools(OpenAI function 포맷) → router → query updater → answer critic 순으로 조립한다.

### 3.1 Retriever tools
- 각 retriever를 OpenAI tools 포맷(name, description, parameters)으로 정의. 예: `movie_info_by_title`, `movies_info_by_actor`, `text2cypher`(fallback).
- description·parameter를 명확히 써야 LLM이 올바른 tool·인자 선택.
- **중요**: LLM은 tool을 직접 호출하지 않고, 어떤 tool을 어떤 인자로 호출할지 "결정"만 함. 실제 호출은 시스템이 수행.
- generic tool `answer_given`: 답이 이미 질문/context에 있으면 추출.

### 3.2 Retriever router
- **handle_tool_calls**: LLM이 고른 tool·인자로 실제 함수 호출. LLM은 여러 tool 호출 결정 가능.
- **continuous query updating**: 질문을 순차 처리하며, 이전 답으로 다음 질문을 재작성(예: "오스카 최다 수상자, 그 사람 생존?" → 1) 최다 수상자 2) 그 사람 생존?). query_update는 더 atomic·specific하게 보완(JSON 출력).
- **routing**: tool_picker_prompt + 질문 + tools로 LLM 호출 → 최적 tool 반환 → handle_tool_calls.

### 3.3 Answer critic
- 모든 답변을 받아 원 질문이 옳게 답해졌는지 검증(LLM은 nondeterministic이라 필요).
- 부족하면 missing 정보를 모을 새 질문 list 반환(JSON), 충분하면 빈 list.

### 3.4 Tying it all together
```python
def main(input):
    answers = handle_user_input(input)
    critique = critique_answers(input, answers)
    if critique:
        answers = handle_user_input(" ".join(critique), answers)
    return chat([{system:main_prompt}, *answers, {user:input}], model="gpt-4o")
```
- main_prompt: prompt에 제공된 정보만 사용, 외부 정보·추측 금지.
- critique는 1회만 수행, 이후에도 불완전하면 그대로 반환하고 LLM이 부족함을 사용자에게 안내.

---

## Summary (핵심 정리)
- agentic RAG는 여러 retrieval agent 중 질문에 필요한 데이터를 찾는 시스템이다.
- 주 인터페이스는 retriever router로, 작업에 가장 적합한 retriever(들)를 찾는다.
- 기본 구성: retriever agents, retriever router, answer critic.
- tool/function-calling을 지원하는 LLM으로 주요 부분을 구현한다.
- retriever agent는 generic·specialized로 나뉘며 필요에 따라 점진적으로 추가한다.
- answer critic은 retriever 답변이 원 질문에 옳게 답하는지 검증한다.
