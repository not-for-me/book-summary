# 03. Building a Foundational Understanding of Knowledge Graph for Intelligent Applications

## 챕터 개요 (3줄 요약)

- knowledge graph가 무엇이며 graph data modeling이 RAG의 retriever를 어떻게 더 효과적으로 만드는지 설명한다.
- 동일 데이터를 RDBMS(ER diagram)와 Neo4j graph(basic/advanced 접근)로 모델링하고 Cypher 쿼리·성능을 비교한다.
- RAG와 Neo4j knowledge graph를 결합한 GraphRAG 흐름을 구축하고, ontology·GDS로 graph를 강화하는 방법을 소개한다.

---

## 1. Understanding the importance of graph data modeling

> graph는 우리가 풀려는 문제에 따라 데이터를 다른 관점으로 보게 강제하며, 이는 제약이 아니라 많은 가능성을 연다.

- 오랫동안 ER(Entity-Relationship) diagram 기반 RDBMS 사고에 익숙해졌으나, 저장 비용 하락으로 새로운 모델링이 가능해졌다.
- graph 모델링은 neural plasticity prism goggles 실험처럼 기존 방식을 unlearn하는 과정이 필요하다.
- 도서관 예시: 책은 카테고리·저자 성으로 배치(index)되지만, 입구의 신간·인기 코너는 RDBMS로 표현하기 어렵다. Neo4j는 multiple label로 이를 쉽게 처리한다.
- Neo4j의 optional flexible schema는 최적이 아닌 데이터 모델로 시작해도 처음부터 다시 만들 필요 없이 점진적으로 튜닝할 수 있게 한다.
- 예제 데이터: 한 person(firstName, lastName)과 그가 거주한 5개 rental(주소, fromTime, tillTime).
- 답하려는 질문: John Doe의 (1) 최신 주소, (2) 첫 주소, (3) 세 번째 주소.

### RDBMS data modeling

> RDBMS는 Person, Address, Person_Address(조인 테이블) 3개 테이블로 모델링하며, 데이터 중복을 피하기 위해 join table을 사용한다.

- join table은 rental 상세와 Person/Address 참조를 담으며, 테이블 분리·변경 시 데이터 마이그레이션 비용이 크다.
- Query 1(최신 주소): end 컬럼이 null인 행으로 판단 — 로직이 SQL에 내장됨.
- Query 2/3(첫/세 번째 주소): ORDER BY start + LIMIT으로 search-sort-filter 패턴 사용.
- RDBMS 데이터 검색의 가장 큰 비용은 join table의 index lookup 비용이며, 데이터가 커질수록 증가한다.

### Graph data modeling: basic approach

> "Person lives at Address" — 명사는 node, 동사는 relationship으로 표현하는 가장 단순한 모델로 ER diagram과 유사하다.

- join table이 HAS_ADDRESS relationship으로 표현되어 index lookup 비용을 줄인다.
- Cypher 스키마: Person/Address에 UNIQUE constraint, Person.name에 index 생성.
- 데이터 로드: `(p)-[:HAS_ADDRESS {start, end}]->(a)` 형태로 5개 주소 연결.
- Query 1: `WHERE r.end is null` — SQL처럼 로직이 쿼리에 남아있다.
- Query 2/3: ORDER BY r.start + LIMIT/SKIP으로 여전히 search-sort-filter 패턴 의존.

### Graph data modeling: advanced approach (consumption approach)

> 데이터 소비 방식에 영향받은 모델로, Person→Rental(FIRST/LATEST), Rental→Rental(NEXT), Rental→Address(HAS_ADDRESS) 구조다.

- Person은 첫(FIRST)·마지막(LATEST) rental에만 연결되고, rental들은 NEXT relationship으로 시간순 연결된다.
- Query 1(최신): `(p)-[:LATEST]->()-[:HAS_ADDRESS]->(a)` — 영어 문장처럼 직관적, search-sort-filter 불필요.
- Query 2(첫): `(p)-[:FIRST]->()-[:HAS_ADDRESS]->(a)`.
- Query 3(세 번째): `(p)-[:FIRST]->()-[:NEXT*2..2]->()-[:HAS_ADDRESS]->(a)` — 그래프 순회로 자연스럽게 표현.

```
(Person)-[:FIRST]->(Rental1)-[:NEXT]->(Rental2)-[:NEXT]->...->(Rental5)<-[:LATEST]-(Person)
   each Rental -[:HAS_ADDRESS]-> (Address)
```

#### 성능 비교 (query profiling)

- Query 1: basic 18 db hits → advanced 12 db hits (메모리 동일 312 bytes).
- Query 2: basic 19 db hits/1,020 bytes → advanced 12 db hits/312 bytes (ordering이 메모리·CPU 소모 유발).
- Query 3: basic 19 db hits/1,028 bytes → advanced 16 db hits/336 bytes.
- advanced 모델은 관계 수가 늘어도 성능이 거의 일정한 반면, basic 모델은 선형적으로 db hits가 증가한다.
- 추가 관점(예: 같은 주소의 rental 추적)이 필요하면 NEXT_RENTAL 관계만 추가하면 되며, RDBMS로는 표현이 어렵다.
- 좋은 graph data model은 RAG의 retriever를 더 효과적으로 만들어 관련 데이터를 빠르고 쉽게 검색한다.

---

## 2. Combining the power of RAG and Neo4j knowledge graphs with GraphRAG

> retriever가 활용하는 data store의 능력이 검색 정보의 유용성·속도·효과를 좌우하며, 여기서 graph가 큰 역할을 해 GraphRAG가 탄생했다.

- Neo4j는 데이터를 node·relationship의 property graph로 저장해 직관적 저장·검색과 RAG retriever용 data store 역할을 한다.
- GraphRAG는 Microsoft 연구(From Local to Global: A Graph RAG Approach to Query-Focused Summarization)로 정립되었다.

### GraphRAG workflow (Figure 3.11)

- 사용자 prompt → LLM API → Neo4j에서 관련 정보 검색 → prompt와 결합 → LLM API → 정확하고 맥락 풍부한 응답 생성.
- Neo4j와 RAG 결합으로 도메인 맥락의 relevance를 향상시킨다.

### Building a knowledge graph for RAG integration (Movies/Plots 예시)

> Neo4j Python Driver와 pandas로 IMDb 영화·plot의 간단한 knowledge graph를 구축한다(하드코딩 데이터).

- 연결 설정: `bolt://localhost:7687`, username/password로 GraphDatabase 연결.
- Movie 노드(title, year)와 Plot 노드(description) 생성 후 `(m:Movie)-[:HAS_PLOT]->(p:Plot)` 관계 생성.
- Cypher로 영화·plot 조회: `MATCH (m:Movie)-[:HAS_PLOT]->(p:Plot) RETURN m.title, m.year, p.description`.

### Integrating RAG with Neo4j knowledge graph

- User input: prompt(예: "The Matrix") 입력.
- Query generation: prompt로 Cypher 쿼리 생성(`WHERE m.title CONTAINS '{prompt}'`).
- Data retrieval: Cypher 실행해 영화 plot 등 관련 데이터 fetch.
- RAG model processing: 검색 데이터를 원본 prompt와 결합해 RAG 모델에 전달.
- Response generation: enriched prompt로 응답 생성(beam search, temperature 0.7, top_k 50, top_p 0.9 등 파라미터 사용).
- 주의: numpy < 2 버전에서 동작하며, numpy > 2 사용 시 가상환경에 numpy==1.26.4 설치 권장.

---

## 3. Enhancing knowledge graphs

> 단순 graph를 더 효과적인 knowledge graph로 만드는 두 가지 핵심 접근을 소개한다(후속 장에서 심화).

- Ontology development: graph의 구조·내용을 정의하여 데이터와 연결을 직관적으로 설명하고, 데이터셋 간 일관성·확장성을 유지(Chapter 5에서 영화 graph 강화).
- Graph Data Science(GDS): link prediction이나 community detection으로 기존 데이터 기반의 추론 관계를 생성해 graph 지능을 강화(Chapter 10에서 KNN similarity·community detection 활용).

---

## Summary (핵심 정리)

- RDBMS와 Neo4j graph 모델링을 비교하여, consumption 기반 advanced graph 모델이 성능·비용 면에서 우수함을 query profiling으로 확인했다.
- RAG와 Neo4j knowledge graph를 결합한 GraphRAG 흐름을 구축하고 통합 워크플로(input→query→retrieval→processing→generation)를 다뤘다.
- 다음 Part 2에서는 Haystack과 Neo4j를 통합하여 AI 기반 검색 기능을 구축하는 방법으로 이어진다.
