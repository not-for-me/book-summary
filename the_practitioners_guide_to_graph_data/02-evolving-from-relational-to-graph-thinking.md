# 02. Evolving from Relational to Graph Thinking

## 챕터 개요 (3줄 요약)
- relational vs graph 기술 선택의 핵심 질문 3가지(graph가 더 나은가 / 데이터를 graph로 어떻게 볼까 / schema 모델링)를 다룬다.
- graph theory의 기본 용어(vertex, edge, adjacency, neighborhood, distance, degree)와 supernode 개념을 정의한다.
- 시각적 schema를 코드로 옮기는 **Graph Schema Language(GSL)** — vertex/edge label, property, direction, multiplicity — 를 소개한다.

---

## 1. Relational vs Graph: 무엇이 다른가

> relational은 relational algebra로 entity 저장·조회에, graph는 graph theory로 relationship 저장·조회에 최적화되어 있다.

- relational(Oracle/MySQL/PostgreSQL)은 relational algebra, graph는 graph theory 기반 — feature 비교가 아니라 근본 수학 이론이 다름.
- 둘 다 entity·relationship 표현 가능하나, 한쪽에 최적화됨.
- 비교 시 주관적 기준인 **ease of use, maintainability**에 주목.

---

## 2. 러닝 예제 데이터 (financial services)

> 5명의 customer가 account/loan/credit card를 공유하는 데이터로 relational·graph 모델을 모두 구성한다.

- 4 entity: customers, accounts, loans, credit cards.
- account·loan은 공유 가능(many-to-many), credit card는 customer 1명 전용.
- parent-child(Michael-Maria), sole user(Rashika), partners(Jamie-Aaliyah) 유형.

---

## 3. Relational Data Modeling (ERD)

> relational에서 entity는 table, attribute는 column이 되고, link는 별도 table 또는 foreign key로 저장된다.

- entity(추적 대상 객체) / attribute(속성).
- ERD의 diamond = 연결, n:m 표기 = many-to-many.
- 한계: 연결을 또 다른 table(entity)로 표현 → 데이터의 connectedness를 이해하기 어려운 mental hurdle.

---

## 4. Concepts in Graph Data — 기본 요소

> graph는 vertex(개념/entity)와 edge(relationship)로 구성된다.

- **Graph**: vertex와 edge로 이루어진 데이터 표현.
- **Vertex**: 개념·entity. (이 책은 distributed 맥락 때문에 'node' 대신 vertex 사용)
- **Edge**: vertex 간 relationship/link.
- 예: customer가 account를 owns, loan을 owes, credit card를 uses.

---

## 5. Adjacency, Neighborhood, Distance

> 데이터가 연결되었는지(if)를 표현하는 핵심 용어들.

- **Adjacency**: 두 vertex가 edge로 연결되면 adjacent.
- **Neighborhood N(v)**: v에 adjacent한 모든 vertex. 2 edge 떨어지면 second neighborhood.
- **Distance**: 한 vertex에서 다른 vertex까지 walk해야 하는 edge 수. dist(Michael, cc_17)=1.

---

## 6. Degree (in-degree / out-degree)와 Supernode

> degree는 데이터가 얼마나 잘(how well) 연결되었는지를 나타낸다.

- **Degree**: vertex에 incident한 edge 수. degree 1 vertex = leaf.
- **In-degree / Out-degree**: 들어오는/나가는 edge 수.
- 고-degree vertex = hub/영향력 큰 entity. edge > 100,000 = **supernode** (성능 이슈, Ch9에서 상세).

---

## 7. Graph Schema Language (GSL) — labels & properties

> 시각적 graph schema를 코드로 번역하기 위한 이 책의 교육용 언어.

- **Vertex label**: 같은 attribute·relationship을 공유하는 객체 class (원으로 표기).
- **Edge label**: vertex label 간 relationship 타입 (이름 붙은 선).
- **Property**: vertex/edge label의 속성(=relational의 attribute).
- 용어 구분: 데이터는 vertex/edge, schema는 vertex label/edge label.

---

## 8. Edge Direction (domain / range)

> edge 방향은 데이터를 subject-predicate-object로 서술하는 방식에서 나온다.

- **Directed**: 한 방향. **Bidirectional**: 양방향(graph theory의 undirected와 동일).
- **Domain**: edge가 시작하는 vertex label(subject). **Range**: edge가 끝나는 vertex label(object).
- **Self-referencing**: domain과 range가 같은 vertex label (예: family 관계, parent-child 재귀 관계).

---

## 9. Multiplicity (set vs collection)

> 대부분 graph DB의 edge label은 many-to-many이며, 인접 vertex 그룹이 set인지 collection인지로 multiplicity를 표현한다.

- **Set**: unique 값만 저장 → 두 vertex 간 edge 최대 1개 (GSL single line).
- **Collection**: 중복 허용 → 여러 edge 가능 (GSL double line + 구별 property 필요, 예: role).
- cardinality = 실제 집합 크기, multiplicity = 허용 가능 크기 범위.
- 시간(time) 모델링 시: 최신 edge만 = set, 전체 이력 = collection.

---

## 10. 모델링과 분석의 구분

> graph schema 설계와 graph data 분석을 혼동하지 말 것 — "pie chart vs foreign key constraint"와 같은 차이.

- graph는 conceptual model이 곧 physical model → 별도 physical 모델링 불필요(mental model→storage 거리가 짧음).
- application(거리 계산 등 분석)과 schema 설계(vertex/edge label)는 별개 개념.
- relationship이 데이터에서 가장 중요하면 graph technology가 정답.

---

## Summary (핵심 정리)
- graph 도입의 3대 질문: graph가 relational보다 나은가 / 데이터를 graph로 어떻게 볼까 / schema를 어떻게 모델링할까.
- graph theory 핵심 용어: vertex, edge, adjacency, neighborhood, distance, degree(in/out), leaf, supernode(>100k edges).
- **GSL** 5요소: vertex label, edge label, property, direction(domain→range, directed/bidirectional/self-referencing), multiplicity(set=single line, collection=double line).
- graph는 conceptual model = physical model이라 mental model에서 storage까지 거리가 짧다.
- schema 설계와 data 분석은 별개 — 혼동하지 말 것. relationship이 핵심이면 graph가 답.
