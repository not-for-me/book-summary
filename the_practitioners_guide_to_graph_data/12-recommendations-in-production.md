# 12. Recommendations in Production

## 챕터 개요 (3줄 요약)
- 실시간 추천의 확장 한계(branching factor + supernode)를 **shortcut edge**(다중 hop 쿼리 결과를 직접 edge로 저장)로 해결한다.
- 별도 환경에서 NPS 쿼리를 movie별로 data parallelism으로 precompute하고, batch computation과의 trade-off를 비교한다.
- production schema(rated=time clustering, recommend=score clustering)와 limit(x)+clustering key로 빠른 추천 쿼리를 구현하고, edge partition 수로 성능을 추론한다.

---

## 1. 확장 실패 원인 — branching factor & supernode

> collaborative filtering 쿼리가 production에서 너무 느린 이유.

- user→movie→user→movie walk가 traverser를 지수적으로 fork (branching factor, Ch6).
- 추천의 supernode 2종: **superuser**(거의 모든 콘텐츠 평가), **superpopular content**(대부분 user가 평가).
- pathfinding(Ch9)과 달리 추천에선 popularity가 trending 신호라 제거가 아닌 활용 필요.

---

## 2. Shortcut Edge

> vertex a→n 다중 hop 쿼리 결과를 a→n 직접 edge로 미리 저장.

- **recommend** edge: movie를 NPS 기준 추천 movie에 직접 연결 → 위험한 구간에 다리를 놓음.
- user가 아닌 movie에서 시작 → user의 최근 평점에 대한 즉각 추천 가능.

---

## 3. Pruning — shortcut edge 계산 제한

> 무엇을·얼마나 자주 precompute할지가 핵심.

- **score threshold**: 점수 임계 이상만 (NPS는 고정 임계 없음).
- **hard limit**: 총 edge 수 제한(예: 상위 1,000) → 디스크 예측·쿼리 용이. 이 책 채택.
- **domain knowledge filter**: genre/배우/trend 등으로 맞춤화.
- 업데이트 고려: 변경된 콘텐츠만 재계산, 성공한 추천(클릭) 반영, robust(작고 결정적·반복가능) 파이프라인.

---

## 4. Shortcut Edge 계산 — schema & 쿼리

> 별도 환경에 movie/user/rated(+MV)만 로드해 movie별 상위 1,000 추천 계산.

```
g.withSack(0.0).V().has("Movie","movie_id",movie_id).
  aggregate("originalMovie").
  inE("rated").has("rating",P.gte(4.5)).outV().
  outE("rated").choose(...sack +1/-1...).inV().
  where(without("originalMovie")).
  group().by().by(sack().sum()).unfold().
  order().by(values,desc).limit(1000).
  project("original","recommendation","score")...toList()
```

- Ch10 쿼리에서 3가지 변경: movie에서 시작, limit(1000), [original/recommend/score] list 생성.
- 예: Aladdin(588) → Lion King, Shawshank, Beauty and the Beast 등.

---

## 5. Data Parallelism vs Batch Computation

> movie별 추천은 독립적이라 작은 쿼리로 분할 가능(data parallelism).

- **data parallelism**: movie_id를 N개 list로 나눠 프로세서별 동기 계산. 선택적 업데이트·재시작 용이, 시작점 적을 때 유리.
- **batch computation**: 공유 계산 활용(같은 reviewer 중복 walk 회피), 전체 재계산. 메모리 多, 동시 transactional 워크로드 방해 가능 → 별도 analytical DC에서 실행 후 operational DC로 복제.
- 이 책은 data parallelism + transactional 채택(시작점이 수천 개로 적음).

---

## 6. Production Schema & 로딩

> rated는 time clustering(최근 접근), recommend는 score clustering(상위 접근).

```
schema.edgeLabel("rated")...clusterBy("timestamp",Text,Desc)...
schema.edgeLabel("recommend").from("Movie").to("Movie").
  clusterBy("nps_score",Double,Desc).create()
```

- shortcut edge CSV(out_movie_id, in_movie_id, nps_score) → dsbulk로 로딩.

---

## 7. 추천 쿼리 — limit(x) + clustering key

> clustering key 정렬 + limit(x)가 disk 상위 N row만 읽어 매우 빠름.

```
// Query 1: 최근 1개 평점의 상위 3 추천
g.V().has("User","user_id",694).
  outE("rated").limit(1).inV().      // 최근 평점(time 정렬)
  outE("recommend").limit(3)...       // 상위 3(score 정렬)

// Query 2: 최근 3개 평점 각각의 상위 1 추천
// Query 3: 최근 3개 평점 각각의 상위 3 (local() scope)
```

- limit는 partition의 첫 N row 선택 → Cassandra adjacency list가 빠른 이유.
- Query 3는 local() scope로 traverser별 3개씩, group으로 중복 score 합산.

---

## 8. 성능 추론 — edge partition 수

> 쿼리 속도는 접근하는 edge partition 수와 데이터 connectivity에 비례한다.

- **Query 1**: edge partition **2개**(user rated + 1 movie의 recommend) → 가장 빠름.
- **Query 2/3**: edge partition **4개**(user rated + 3 movie 각각의 recommend).
- 성능 = 분산 partition 관리 + branching factor 계획의 균형. shortcut edge가 둘 다 완화.
- 추가 고려: 본 영화 제외 필터, result set 크기(1,000 precompute), 무한 스크롤 streaming.

---

## Summary (핵심 정리)
- 실시간 추천의 branching factor·supernode 문제를 shortcut edge(다중 hop을 직접 edge로 precompute)로 해결.
- pruning 3방식: score threshold, hard limit(상위 1,000 채택), domain knowledge filter.
- precompute는 movie별 독립이라 data parallelism으로 분할(vs batch는 공유계산·전체재계산, 별도 DC).
- production schema: rated=time clustering, recommend=score clustering. limit(x)+clustering key로 빠른 쿼리.
- 성능은 접근 edge partition 수에 비례 — Query1(2개)이 Query2/3(4개)보다 빠름. shortcut edge가 BF·supernode 동시 완화.
