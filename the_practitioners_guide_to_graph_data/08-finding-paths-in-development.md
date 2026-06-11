# 08. Finding Paths in Development

## 챕터 개요 (3줄 요약)
- path를 trust 정량화에 활용하는 관점(소셜미디어/수사/물류)과 shortest path 문제 유형(shortest/single-source/all-pairs), DFS/BFS 기초를 다룬다.
- Bitcoin OTC trust network를 예제로 vertex(Address)·edge(rated, trust property) schema와 neighborhood 탐색 쿼리를 구현한다.
- repeat()/until() + limit + BFS(barrier step)로 shortest path를 찾고, sack()으로 trust 합산하는데, "긴 path일수록 trust 합 높음" 문제를 노출(Ch9 복선).

---

## 1. Trust = 거리 — 3가지 예제

> 개념 간 거리(path)가 trust를 정량화한다.

- 소셜미디어: 공유 connection의 quantity·quality로 신규 요청 수락 판단.
- 범죄 수사: 데이터 소스를 통합해 두 인물 사이 path로 사건 구성.
- 물류: 적은 transfer(짧은 path)일수록 분실 위험↓, 신뢰↑.

---

## 2. Shortest Path 기본 개념

> shortest path = 두 vertex를 잇는 최소 length(edge 수) path.

- **Path**: 연속된 edge sequence. **Length**: edge 수. **Distance**: shortest path의 length.
- 3 유형: **shortest**(A→B), **single-source**(A→전체), **all-pairs**(임의 두 vertex). 이 장은 shortest.
- 알고리즘보다 "어떤 path 문제인지" 먼저 파악할 것.

---

## 3. DFS vs BFS

> 모든 pathfinding 알고리즘의 기반.

- **DFS**: 한 branch를 끝까지 깊게 탐색 후 backtrack. LIFO stack (수직).
- **BFS**: 같은 depth(neighborhood) 전부 탐색 후 다음 depth. FIFO queue (수평).
- BFS면 stopping 조건 만족하는 첫 traverser가 shortest path 보장.

---

## 4. Bitcoin Trust Network — 데이터 & schema

> Bitcoin OTC의 who-trusts-whom 평점 데이터(SOURCE,TARGET,RATING[-10,10],TIME).

```
schema.vertexLabel("Address").partitionBy("public_key",Text).create();
schema.edgeLabel("rated").from("Address").to("Address").
  clusterBy("trust",Int,Desc).property("datetime",Text).create()
```

- address=Bitcoin public key, wallet=private key 모음.
- 5,881 vertex / 35,592 edge. Louvain community detection으로 trust 커뮤니티 시각화.

---

## 5. Neighborhood 탐색 & aggregate/where(without)

> 2nd neighborhood이지만 1st에 없는 address를 찾아 pathfinding 예제 쌍 구성(test-driven).

```
dev.V().has("Address","public_key","1094").aggregate("x").
  out("rated").aggregate("x").
  out("rated").dedup().where(without("x")).values("public_key")
```

- **dedup()**: 중복 제거(2nd neighborhood 876→613 unique).
- **aggregate("x") + where(without("x"))**: 1st neighborhood 제거(relational의 right outer self join 유사).
- sample(1)로 무작위 선택 → 1337 (vs 시작 1094).

---

## 6. Lazy vs Eager & Barrier Step

> Gremlin은 기본 lazy stream 처리지만, barrier step에서 eager(BFS 유사)로 전환된다.

- **Lazy**: 값이 필요할 때까지 평가 지연. **Eager**: 변수 바인딩 즉시 평가.
- **Barrier step**(dedup/aggregate/count/order/group/groupCount/cap/iterate/fold): 모든 traverser가 그 지점까지 완료될 때까지 대기 → BFS처럼 동작.

---

## 7. Fixed/Any Length Path & BFS로 shortest 보장

> repeat().until()는 NoOpBarrierStep을 주입해 eager(BFS) → 첫 traverser가 shortest path.

```
dev.V().has(...,"1094").
  repeat(out("rated")).
  until(has(...,"1337")).
  limit(1).                  // BFS라 첫 traverser = shortest path
  path().by("public_key")...
```

- `repeat().until()`만 쓰면 all-paths 탐색 → 30s timeout.
- explain()으로 NoOpBarrierStep 확인 → eager 평가 검증. limit(1)로 shortest, limit(15)로 상위 15 shortest.

---

## 8. sack()으로 trust 합산 & 문제 노출 (Ch9 복선)

> 각 traverser의 sack에 edge trust를 누적하는데, 긴 path일수록 trust 합이 커지는 모순.

```
dev.withSack(0.0).V().has(...,"1094").
  repeat(outE("rated").sack(sum).by("trust").inV().simplePath()).
  until(has(...,"1337")).limit(15).
  order().by(sack(),decr).
  project("path_information","vertices_plus_edges","total_trust")...
```

- outE()로 edge에 멈춰 trust 수집, sack(sum).by("trust")로 누적, simplePath()로 cycle 제거.
- 결과: 가장 trust 높은 path가 가장 긴 path → 의미 없음. edge weight **normalization** + shortest **weighted** path 필요(Ch9).

---

## Summary (핵심 정리)
- path 거리는 trust를 정량화 — 소셜/수사/물류에서 공통.
- shortest path 3 유형(shortest/single-source/all-pairs), 기반은 DFS(stack)·BFS(queue).
- Bitcoin OTC schema: Address vertex + rated edge(trust, datetime).
- Gremlin은 lazy지만 barrier step에서 eager(BFS); repeat().until()는 BFS라 limit(1)이 shortest path 보장. all-paths는 timeout.
- sack()으로 edge weight 누적 가능하나, 단순 합산은 긴 path가 유리해 무의미 → normalization + weighted shortest path(Ch9).
