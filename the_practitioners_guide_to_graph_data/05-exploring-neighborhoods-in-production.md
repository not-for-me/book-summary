# 05. Exploring Neighborhoods in Production

## 챕터 개요 (3줄 요약)
- DataStax Graph = Apache Cassandra 위의 graph로, "읽을 방식대로 쓴다"는 분산 데이터 패러다임에서 primary/partition key, clustering column, materialized view를 다룬다.
- edge의 disk 표현(edge list/adjacency list/matrix)과 reverse traversal에 필요한 materialized view, denormalization·indexing 최적화를 설명한다.
- 개발→운영 전환을 위한 10대 data modeling 팁(Rule of Thumb #7~#10 포함)과 dsbulk 로딩, production Gremlin 쿼리를 완성한다.

---

## 1. Primary Key & Partition Key

> primary key는 데이터를 유일하게 식별하고, partition key(primary key의 첫 요소)는 분산 환경에서 데이터 위치를 결정한다.

- full primary key로 읽는 것이 가장 빠른 접근.
- partitionBy()로 partition key 지정 → 데이터가 어느 host에 쓰일지 결정.
- 의미: customer_id가 곧 식별자이자 쿼리 시작점이자 데이터 locality 결정자.

---

## 2. Partition 전략 — access pattern vs unique key

> partition key 설계는 traversal latency와 query 유연성 사이의 trade-off이다.

- **access pattern 기준**(customer_id를 모든 vertex의 partition key로): 데이터 colocation → latency 최소화. 단 shared account가 분리되어 reverse 조회 불가.
- **unique key 기준**(각 label 고유 키): query 유연성 최대. 단 연결 데이터 walk 시 머신 간 점프로 latency↑. → C360 예제는 이 방식 채택.
- graph partitioning은 NP-complete 문제 — ERD 변환처럼 간단하지 않음.

---

## 3. Edge 표현 3가지 자료구조

> Cassandra는 distributed adjacency list로 edge를 저장한다.

- **edge list**: (from,to) 쌍 목록. 가장 압축적이나 특정 vertex edge 찾으려면 전체 스캔.
- **adjacency list**: key=vertex, value=인접 vertex 목록. 인덱싱된 접근 + 스캔 최소화 (Cassandra 채택).
- **adjacency matrix**: V×V 표. walk는 빠르나 공간 과다.

---

## 4. Clustering Column

> clustering column은 disk 상 데이터 정렬 순서를 결정하며 edge label primary key의 마지막 요소가 된다.

- edge label: from(vertex)=partition key, to(vertex)=clustering column.
- collection(이중선 edge): property를 clustering key로 → 두 vertex 간 다중 edge 허용.
- vertex의 outgoing edge는 vertex와 같은 머신에 colocate, incoming vertex의 primary key로 정렬.

---

## 5. Materialized View — bidirectional edge

> edge는 partition key를 알아야 접근 가능하므로, 기본적으로 reverse 방향 traversal이 불가하다.

- Ch5-1 에러: `out("owns").in("withdraw_from","deposit_to")` → reverse walk에 index 필요.
- **Materialized view**: 다른 primary key 구조로 데이터 복제본을 생성·유지 (앱이 수동 중복 기록 불필요).
- inverse() materializedView 생성 시 partition/clustering key가 뒤집힌 테이블 생성 → bidirectional edge 확보.

---

## 6. Graph Data Modeling 201 — denormalization & index (RoT #7~#8)

> 데이터 중복을 감수하고 읽기 성능을 높이는 denormalization과, 걷는 방향에 맞춘 최소 index 설계.

- **#7** property를 edge/vertex에 중복 저장(denormalization) → 처리할 요소 수 감소. 예: timestamp를 edge에도 저장해 최근 20 edge만 subselect.
- **#8** 걷고 싶은 방향이 필요한 index(materialized view)를 결정. 쿼리를 schema에 매핑해 "역방향 walk" 지점을 찾음.
- `schema.indexFor(traversal).analyze()`로 자동 index 추천 (수동 매핑과 동일한 일).

---

## 7. 운영 전환 팁 #9~#10 & dsbulk 로딩

> 데이터 먼저 로드 후 index 적용, 운영에 필요한 edge·index만 유지.

- **#9** load data → then apply indexes (로딩 속도↑, blue-green 배포 패턴).
- **#10** production 쿼리에 필요한 edge·index만 유지(공간·시간 절약).
- **dsbulk**: `dsbulk load -url ... -g graph -v VertexLabel -header true` (vertex), edge는 `-e label -from -to`. CSV header가 곧 매핑 설정.

---

## 8. Production Gremlin — in() vs inE()

> dev → g 운영 source 전환 + materialized view·clustering key 활용으로 처리 데이터를 최소화한다.

```
g.V().has("Customer","customer_id","customer_0").
  out("owns").
  inE("withdraw_from","deposit_to").   // E 추가: edge 접근 + MV 사용
  order().by("timestamp",desc).limit(20).
  outV().values("transaction_id")
```

- `in()` → `inE()` 한 글자 변경으로 edge를 disk 정렬 순서대로 처리(최근 20 edge만 walk).
- Query 2도 inE()+timestamp range, Query 3는 모두 outgoing edge라 MV 불필요(source만 g로 교체).

---

## Summary (핵심 정리)
- DataStax Graph = Cassandra 위 graph, "읽을 방식대로 쓴다". full primary key 접근이 가장 빠름.
- partition key = locality, clustering column = 정렬 순서. partition 전략은 access pattern(colocation/latency↓) vs unique key(유연성) trade-off, NP-complete.
- edge는 adjacency list로 저장. reverse traversal엔 inverse materialized view 필요.
- 10대 팁: #1~6(Ch4 모델링) + #7 denormalization, #8 방향 기반 index, #9 load→index, #10 필요한 것만 유지.
- 운영 Gremlin: `in()`→`inE()`로 MV·clustering key 활용해 데이터 처리량 최소화, dsbulk로 CSV 로딩.
