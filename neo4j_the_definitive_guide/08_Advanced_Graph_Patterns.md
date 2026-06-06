# 08. Advanced Graph Patterns

## 챕터 개요 (3줄 요약)

- 고급 Cypher 기능인 서브쿼리(`CALL`)와 동시 트랜잭션(`IN CONCURRENT TRANSACTIONS`), 세분화된 관계 타입을 활용해 메모리 효율과 성능을 높이는 방법을 다룬다.
- 엔티티 해결(entity resolution) 문제를 엔티티 그룹(EntityGroup)·융합 엔티티(fused entity)·정량화 경로 패턴(QPP, Quantified Path Patterns)으로 모델링한다.
- 라벨 기반(LBAC)과 속성 기반(PBAC) 접근 제어를 비교하며 보안 모델링 관점을 제시한다.

---

## 1. Subqueries

> Cypher 서브쿼리는 외부 쿼리의 각 행마다 자체 스코프에서 실행되어 메모리 오버헤드를 줄이는 중첩 쿼리다.

- `CALL` 서브쿼리: 외부 쿼리에서 들어온 행마다 독립 스코프로 실행되며, 행 처리 후 자료구조를 유지하지 않아 메모리를 절약한다 — 1,800만 노드/1억 관계 그래프에서 효과가 크다.
- 유니온 후 처리(post-union processing): `UNION` 결과를 다시 집계할 때 서브쿼리가 유용하다(예: 긴 트랙 아티스트 수 + 500개 이상 플레이리스트에 든 아티스트 수 합산).
- `COUNT { ... }` 같은 존재/개수 서브쿼리로 조건을 간결하게 표현한다.
- 동시 트랜잭션(`CALL {} IN CONCURRENT TRANSACTIONS`): 기본 단일 코어 실행에 `CONCURRENT`를 더해 배치를 병렬 실행, 지정/가용 CPU 코어를 활용해 적재 속도를 높인다.

```
LOAD CSV ... AS row
CALL (row) {
  MATCH (t:Track {id: row.track_id})
  MATCH (a:Artist {id: row.artist_id})
  MERGE (t)-[:ARTIST]->(a)
} IN CONCURRENT TRANSACTIONS
```

---

## 2. Fine-Grained Relationship Types

> 관계 타입의 세분화는 노드 차수(degree)와 직결되며, Neo4j는 타입·방향별 차수를 내부적으로 추적해 쿼리 성능을 최적화한다.

- 노드 차수(node degree): 노드에 연결된 관계 수로, 상호 연결 정도를 나타내며 쿼리 성능에 핵심적이다.
- Neo4j는 관계 타입과 방향별로 차수를 별도 추적한다(예: 들어오는 `HAS_TRACK` 차수 9).
- 관계 타입을 적절히 세분화하면 플래너가 차수 정보를 활용해 더 선택적인 순회를 한다.
- 동적 라벨/타입(dynamic labels and types)으로 런타임에 라벨·관계 타입을 변수로 지정할 수도 있다.

---

## 3. Modeling Resolved Entities

> 중복·오타로 같은 엔티티가 여러 개 생기는 문제는 엔티티 그룹으로 묶거나 단일 노드로 융합해 해결한다.

- 문제: 인수한 카탈로그를 급히 병합하며 같은 아티스트(예: Guns N' Roses)가 철자 변형으로 중복 생성되어 검색 품질이 떨어진다.
- 엔티티 그룹(entity groups): 원본을 구분 유지해야 해 병합할 수 없을 때, `EntityGroup` 라벨 노드로 "같을 수 있음"을 표현하고 시각화 도구가 묶어 보여준다.
- 융합 엔티티(fused entities): 실제로 해결(resolve)해 여러 노드와 그 관계를 하나의 노드로 병합한다.
- 융합 방식 선택은 출처 보존 필요성과 사용 사례에 따라 달라진다.

---

## 4. Quantified Path Patterns & Security Modeling

> QPP는 반복되는 경로 부분을 `UNION` 없이 간결하게 매칭하며, 보안 모델링은 라벨 기반과 속성 기반 접근 제어 중 선택한다.

- 정량화 경로 패턴(QPP, Quantified Path Patterns): 경로의 반복 부분을 괄호로 추출하고 수량자(quantifier)로 반복 횟수를 지정해, 가변 길이 경로를 간결·표현력 있게 매칭한다 — 엔티티 해결 사용 사례에 적합하다.
- 라벨 기반 접근 제어(LBAC, Label-Based Access Control): 노드 라벨·관계 타입 기준으로 접근을 제한해 RBAC를 높은 수준에서 손쉽게 적용한다.
- 속성 기반 접근 제어(PBAC, Property-Based Access Control): 노드/관계의 속성 수준 가시성 규칙으로 더 동적·세밀한 접근 정책을 정의한다.
- 두 방식은 보안 요구의 세분화 수준에 따라 선택·조합한다.

---

## Summary (핵심 정리)

- `CALL` 서브쿼리와 `IN CONCURRENT TRANSACTIONS`는 대규모 그래프에서 메모리 효율과 적재/조회 성능을 크게 높인다.
- 엔티티 해결은 출처 보존 여부에 따라 엔티티 그룹 또는 융합 엔티티로 모델링하며, QPP로 반복 경로를 간결히 매칭한다.
- 보안 모델링은 라벨 기반(LBAC)과 속성 기반(PBAC) 접근 제어를 요구 세분화 수준에 맞춰 선택한다.
