# 03. Getting Started: A Simple Customer 360

## 챕터 개요 (3줄 요약)
- graph data의 출발점 use case인 **Customer 360(C360)** 를 정의하고, 동일 데이터를 relational(Postgres/SQL)과 graph(DataStax/Gremlin)로 각각 구현한다.
- 같은 4가지 C360 질문을 SQL의 SELECT-FROM-WHERE(join 다수)와 Gremlin의 WHERE-JOIN-SELECT(traversal)로 비교한다.
- data modeling, relationship 표현, query language 측면에서 relational vs graph의 장단점과 선택 기준을 정리한다.

---

## 1. C360 — graph data의 기초 use case

> C360는 customer를 중심에 두고 핵심 entity와의 relationship을 모델링하는, graph thinking 도입의 출발점이다.

- 분산된 silo 데이터를 통합해 customer 중심으로 빠르게 retrieval하는 것이 목표.
- "data lake에서 낚시 vs 저녁 주문" 비유 — 통합(integration)보다 accessibility가 문제.
- personalization으로 매출 최대 10%↑ (BCG). 예: Baidu × KFC 주문 추천.

---

## 2. Relational 구현 — ERD → 물리 모델

> entity는 table, 1:n은 foreign key, n:m은 join table로 표현한다.

- 4 entity table: Customer, Account, Loan, CreditCard.
- CreditCard는 1:n → customer_id를 FK로 보유.
- Owns/Owes는 join table — 두 FK의 compound primary key로 m:n 표현.

---

## 3. Relational C360 쿼리 (SQL)

> 깊은 질문일수록 LEFT JOIN이 누적되어 SQL이 복잡해진다.

```
SELECT-FROM-WHERE pattern
- credit cards: Customers LEFT JOIN CreditCards (1 join)
- accounts:     Customers JOIN Owns JOIN Accounts (join table 경유)
- "everything": 6개 table 전부 LEFT JOIN
```

- 1:n은 join 1번, m:n은 join table 경유로 join 2번.
- "고객에 대해 다 알려줘" = 6 table 조인 → 데이터 추적이 어려움.

---

## 4. Graph 구현 — schema & 데이터 삽입 (Gremlin/DataStax)

> conceptual model에서 graph model로의 전환이 짧다 — 개념 모델이 곧 물리 모델.

```
schema.vertexLabel("Customer").partitionBy("customer_id",Text).property("name",Text).create();
schema.edgeLabel("owns").from("Customer").to("Account").property("role",Text).create();
g.addV("Customer").property("customer_id","customer_0").property("name","Michael").next();
g.addE("owns").from(michael).to(acct_14).property("role","primary").next();
```

- 4 vertex label + 3 edge label(owns/uses/owes), owns에 role property.
- addV는 full primary key 필수, next()는 terminal step.

---

## 5. Graph traversal & traversal source

> graph query = traversal(데이터를 걷는 것). dev(개발, index 불필요) / g(production) source 사용.

- **Graph traversal**: 정해진 순서로 vertex·edge를 방문하는 반복 과정.
- **Traversal source**: 탐색 대상 데이터 + 전략. dev = 개발용, g = 운영용.
- production 성능 위해 full primary key로 특정 vertex에서 시작 권장.

---

## 6. Graph C360 쿼리 — WHERE-JOIN-SELECT

> Gremlin은 SQL의 역순 — 시작 위치(WHERE) → relationship으로 JOIN → 반환 데이터 SELECT.

```
dev.V().has("Customer","customer_id","customer_0"). // WHERE
    out("uses").                                     // JOIN
    values("cc_num")                                 // SELECT
```

- as()/select()/by()로 걸어온 데이터에 라벨 붙이고(=빵부스러기) 여러 값 반환.
- "다 알려줘" = out().elementMap() 로 first neighborhood 전체 반환 → SQL 대비 훨씬 짧음.

---

## 7. Relational vs Graph 비교 (modeling/relationship/query)

> 성숙도는 relational, relationship 표현·표현력은 graph가 우위.

- **Data modeling**: relational은 문서·자료 풍부(정량 우위), graph는 human 직관과 일치(직관 우위).
- **Relationship 표현**: relational은 넣기 쉬우나 꺼내 reasoning하기 어려움. graph는 human↔machine 전환이 매끄러움.
- **Query**: SQL은 성숙도 우위, 얕은 쿼리는 잘 튜닝된 RDBMS가 빠름. 깊은/중첩 쿼리는 Gremlin이 표현력·성능 우위.

---

## 8. 선택 기준 — Why Not Relational?

> relational은 tabular 데이터에, graph는 complex(깊은 relationship) 데이터에 적합하다.

- 단순 C360 통합만이 목표 → 잘 튜닝된 relational이 빠르고 비용 효율적.
- 데이터 아키텍처의 출발점/확장 기반으로 삼을 것 → graph의 학습 곡선이 장기적으로 더 큰 가치.
- 깊거나 계획되지 않은 쿼리: relational은 table 추가·아키텍처 변경 필요, graph는 schema 보강 + 데이터 삽입만.
- 같은 템플릿이 Business 360(B2B)에도 적용 가능.

---

## Summary (핵심 정리)
- C360는 customer를 중심으로 relationship을 통합하는 graph 도입의 기초 use case (B360로도 확장).
- relational: entity=table, 1:n=FK, m:n=join table. 깊은 쿼리는 LEFT JOIN 누적으로 복잡.
- graph: conceptual=physical model, Gremlin traversal은 WHERE-JOIN-SELECT(SQL의 역순), 코드가 훨씬 간결.
- 비교 — 성숙도/문서는 relational 우위, relationship 표현·표현력·깊은 쿼리 성능은 graph 우위.
- 선택: 단순 통합이면 relational, connectedness 탐색·확장 기반이면 graph.
