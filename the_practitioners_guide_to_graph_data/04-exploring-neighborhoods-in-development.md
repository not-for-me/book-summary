# 04. Exploring Neighborhoods in Development

## 챕터 개요 (3줄 요약)
- C360 예제를 확장해 transaction/vendor를 추가하고, query-driven design으로 property graph 모델링 best practice(vertex vs edge vs property, direction, naming)를 정리한다.
- 3~5 neighborhood를 walk하는 Gremlin 쿼리(order/limit, time window range, groupCount, mutating traversal)를 단계별로 구현한다.
- project/fold/unfold, where(neq()), coalesce()로 query 결과 payload(JSON)를 정교하게 shaping하는 advanced Gremlin을 다룬다.

---

## 1. Graph Data Modeling 101 — vertex vs edge (Rules of Thumb 1~4)

> 쿼리를 subject-verb-object 문장으로 적으면 vertex/edge가 자연스럽게 드러난다.

- **#1** traversal을 시작하는 데이터 → vertex.
- **#2** 개념을 연결하는 데이터 → edge (graph가 필요한 이유).
- **#3** Vertex-Edge-Vertex가 문장처럼 읽혀야 함 ("customer owns account").
- **#4** 명사/개념 → vertex label, 동사 → edge label. (단 "ownership"을 entity로 보면 vertex일 수도)

---

## 2. Edge Direction (Rules of Thumb 5)

> 개발 단계에서 edge 방향은 도메인을 서술하는 방식(subject→object)을 따른다.

- transaction은 동사가 아닌 명사 → **vertex label**.
- 돈의 흐름이 아니라 쿼리 사용 방식으로 방향 결정: "Transactions withdraw_from / deposit_to accounts" → Transaction에서 Account로.
- 방향을 쿼리 관점에 맞추면 Query 1(`in("withdraw_from","deposit_to")`)이 훨씬 쉬워짐.

---

## 3. Property는 언제? (Rules of Thumb 6) & Naming Pitfalls

> 그룹을 subselect하는 데이터는 property로 만든다.

- **#6** 시간순 정렬/필터링 위해 timestamp를 Transaction의 property로.
- naming pitfall: `has` edge label 금지(의미·방향 불명) → 능동 동사(deposit_to/withdraw_from); `id` property 금지(Cassandra 충돌) → {label}_id; casing 일관성(vertex=CamelCase, edge/property=snake_case).

---

## 4. Development Graph Model 완성

> 데이터·쿼리·end user 세 가지에 집중해 query-driven으로 schema를 구축한다.

```
Transaction --withdraw_from/deposit_to--> Account
Transaction --pay--> Loan / Vendor
Transaction --charge--> CreditCard
Transaction{transaction_id, transaction_type, timestamp(ISO 8601 text)}
Vendor{vendor_id, vendor_name}
```

- 모델링은 engineering이자 art — relationship-first 사고로 진화.
- production에서 unbounded/open traversal 지양(보안·성능·유지보수).

---

## 5. Gremlin Query 1 — 최근 20개 transaction

> order().by(...desc) + limit()으로 time 기준 상위 N개를 추출한다.

```
dev.V().has("Customer","customer_id","customer_0").
    out("owns").
    in("withdraw_from","deposit_to").
    order().by("timestamp", desc).
    limit(20).
    values("transaction_id")
```

---

## 6. Gremlin Query 2 — time window range & groupCount

> has(...between(...)) 로 기간 필터, groupCount().by()로 집계, order(local)로 map 정렬.

```
... in("charge").
    has("timestamp", between("2020-12-01T00:00:00Z","2021-01-01T00:00:00Z")).
    out("pay").groupCount().by("vendor_name").
    order(local).by(values, desc)
```

- predicate: eq/neq/gt/gte/lt/lte/between(상한 제외).
- **scope**: local(현재 객체=map 내부) vs global(스트림 전체).

---

## 7. Gremlin Query 3 — 5 neighborhood & mutating traversal

> 다중 neighborhood를 walk해 데이터를 찾고(access) property를 갱신(mutation)한 뒤 검증(validation)한다.

```
// 3b: mortgage 거래에 transaction_type 갱신
... in("withdraw_from").
    filter(out("pay").has("Loan","loan_id","loan_18")).
    property("transaction_type","mortgage_payment").
    values("transaction_id","transaction_type")
```

- hasLabel("Loan")로 vertex 종류 필터, mutating traversal로 graph 갱신.
- 3c: groupCount로 전체 갱신 안 됐는지 검증(mortgage 24 / unknown 47).

---

## 8. Advanced Gremlin — project / fold / unfold

> project()로 다중 key map을 만들고 by() modulator로 각 값을 채운다. Gremlin은 lazy → fold()로 barrier.

```
project("CreditCardUsers","AccountOwners","LoanOwners").
  by(out("uses").in("uses").values("name").fold()).
  by(out("owns").in("owns").values("name").fold()). ...
```

- fold()는 모든 결과를 list로 모으는 barrier (lazy 평가로 첫 결과만 나오는 문제 해결).

---

## 9. where(neq()) & coalesce() — 결과 정제와 try/catch

> as()로 저장한 vertex를 where(neq())로 제외하고, coalesce()로 빈 결과에 대한 try/catch 패턴을 구현한다.

```
... as("michael").
  by(out("uses").in("uses").where(neq("michael")).values("name").
     fold().coalesce(unfold(), constant("NoOtherUsers")).fold())
```

- where(neq("michael")): Michael 자신을 결과에서 제거.
- coalesce(try, catch): 빈 list는 unfold 불가 → catch로 "NoOtherUsers" 주입, 뒤에 fold()로 list 구조 보장.

---

## Summary (핵심 정리)
- 6 Rules of Thumb: 시작점=vertex, 연결=edge, V-E-V는 문장, 명사=vertex/동사=edge, 방향은 subject→object, subselect=property.
- naming: `has`/`id` 지양, 능동 동사 edge label, 일관 casing(vertex CamelCase / edge·property snake_case).
- 모델링 3원칙: 가진 데이터에 집중, query-driven design, end user 중심.
- Gremlin 핵심 step: out/in, order().by(desc), limit, has(between), groupCount().by, scope(local/global), mutating property().
- 결과 shaping: project + by, fold(barrier)/unfold, where(neq())로 제외, coalesce()로 try/catch.
