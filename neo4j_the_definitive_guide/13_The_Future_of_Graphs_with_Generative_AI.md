# 13. The Future of Graphs with Generative AI

## 챕터 개요 (3줄 요약)

- 지식 그래프(knowledge graph)가 생성형 AI(GenAI)의 이상적 동반자인 이유와, 분류체계(taxonomy)·온톨로지(ontology) 같은 조직 원리로 institutional intelligence를 포착하는 방식을 설명한다.
- GraphRAG가 단순 벡터 검색(vector search)보다 풍부한 컨텍스트로 LLM(Large Language Model) 응답을 검증된 사실에 근거하게 하는 원리, 그리고 에이전트형 AI(agentic AI) 아키텍처를 다룬다.
- ElectricHarmony 사례로 GDS·벡터 유사도·자연어 인터페이스를 결합해 자연어 질문으로 플레이리스트를 추천하는 6단계 실전 예제로 책을 마무리한다.

---

## 1. Knowledge Graphs

> 지식 그래프는 실세계 엔티티와 관계에 대한 지식을, 조직 원리(스키마·분류체계·온톨로지)를 갖춰 그래프 데이터베이스에 저장한 것이다.

- 조직 원리(organizing principle): 노드와 관계가 어떻게 구성되는지 규정하며, 단순 스키마부터 추론 엔진(inference engine)을 구동하는 복잡한 온톨로지까지 다양하다.
- 분류체계(taxonomy)는 엔티티를 범주로 묶는 분류 시스템이고, 온톨로지는 더 풍부한 의미 관계를 표현한다.
- 활용: Customer 360(여러 사일로의 고객 데이터를 통합하는 마스터 데이터 관리), 사이버보안, 생명과학(life sciences), 리테일, 범죄 수사.
- LLM의 역할을 이해하면 지식 그래프가 왜 GenAI 애플리케이션의 완벽한 동반자인지 드러난다.

---

## 2. GraphRAG vs. Vector Search

> GraphRAG는 외부·독점 데이터에서 관련 정보를 검색·랭킹해 프롬프트를 보강함으로써, LLM이 사전학습 지식이 아닌 제공된 사실에 근거해 답하게 한다.

- 검색 증강 생성(RAG, Retrieval-Augmented Generation): 사용자 질의와 관련된 핵심 정보·컨텍스트를 외부 소스에서 검색·랭킹해 프롬프트에 더한 뒤 LLM이 답을 생성한다.
- 벡터 검색(vector search): 텍스트 임베딩(embedding, 의미를 수백~수천 차원 숫자 배열로 표현)에 대한 유사도 검색으로 의미적으로 관련된 정보를 찾는 인기 RAG 기법이다.
- GraphRAG는 명시적 관계를 가진 그래프를 활용해 벡터 검색보다 더 풍부하고 정확한 컨텍스트를 제공하며, 환각(hallucination)을 줄인다.
- 고품질 컨텍스트가 LLM 응답 품질을 좌우한다.

```
RAG flow:
  user query -> retrieve relevant facts (vector + graph) -> rank ->
  augment prompt with facts -> LLM generates grounded answer
GraphRAG adds explicit relationships for richer, validated context
```

---

## 3. Agentic AI & Knowledge Graph Creation

> 에이전트형 아키텍처는 LLM에 정보 검색뿐 아니라 행동까지 수행하는 도구(tool)를 부여하며, LLM은 지식 그래프 생성·보강도 돕는다.

- 에이전트형 AI(agentic AI): LLM이 사용자 질의를 신뢰성 있게 답하기 위한 도구 집합으로 계획(plan)을 세우고, 도구를 순차/병렬로 호출해 정보 검색과 행동을 수행한다.
- 지식 그래프 생성(creation): 데이터가 조직 전반에 흩어져 있으면 RAG용 컨텍스트 구성이 어려운데, LLM을 쓰면 비정형 데이터로 지식 그래프를 만들거나 보강하기 쉽다.
- Neo4j LLM Knowledge Graph Builder는 문서·PDF·영상 자막 등을 문서·청크(chunk)의 어휘 그래프(lexical graph)로 변환해 임베딩과 함께 저장한다.

---

## 4. Practical Example: Natural Language Playlist Recommendations

> GDS·벡터 유사도·자연어 인터페이스를 결합해 자연어 질문으로 개인화된 음악 추천을 제공하는 6단계 파이프라인이다.

- Step 1 — GDS 커뮤니티: 12장의 Louvain 알고리즘으로 `CO_OCCURS` 관계 기반 플레이리스트 커뮤니티를 탐지한다.
- Step 2 — LLM 요약·질문 생성: 커뮤니티 플레이리스트의 트랙·아티스트 샘플로 LLM이 텍스트 요약과 예상 질문을 생성한다.
- Step 3 — 벡터화·저장: 요약과 질문을 의미를 담은 벡터 임베딩으로 변환해 Neo4j에 저장한다(사용자가 정확히 같은 표현을 쓰지 않으므로).
- Step 4 — 사용자 질문: "시크한 스타일의 재즈 플레이리스트 추천" 같은 질의를 임베딩해 벡터 유사도 검색으로 가장 관련성 높은 요약/질문을 찾는다.
- Step 5 — 답변 생성: 매칭된 커뮤니티에서 그래프를 다시 순회해 트랙 샘플을 가져와 LLM에 전달, 추천 이유(reasoning)를 포함한 답을 생성한다.
- Step 6 — 마무리: GDS·벡터·LLM·그래프 순회가 조화롭게 결합되어 자연어 추천이 완성된다.

---

## Summary (핵심 정리)

- 지식 그래프는 조직 원리(스키마·분류체계·온톨로지)로 실세계 지식을 구조화해 GenAI에 검증된 컨텍스트를 제공하는 이상적 동반자다.
- GraphRAG는 명시적 관계를 활용해 단순 벡터 검색보다 풍부·정확한 근거를 LLM에 제공하며, 에이전트형 AI는 검색을 넘어 행동까지 수행한다.
- GDS 커뮤니티 탐지·벡터 임베딩·LLM·그래프 순회를 결합하면 자연어 질문으로 개인화 추천을 제공하는 실전 파이프라인을 구축할 수 있다.
