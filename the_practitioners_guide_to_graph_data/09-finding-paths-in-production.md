# 09. Finding Paths in Production

## 챕터 개요 (3줄 요약)
- shortest weighted path(최소 비용 path) 문제를 정의하고, lowest cost·supernode avoidance·global heuristic 등 search 최적화와 A* 알고리즘을 다룬다.
- Bitcoin trust 데이터의 edge weight를 [0,1] shift → log → ×(-1)로 normalize해 "높은 trust = 짧은 거리"로 변환한다.
- group/filter/where/and/sideEffect로 최적화를 적용한 production shortest weighted path 쿼리를 Gremlin으로 구현한다.

---

## 1. Shortest Weighted Path 정의

> 두 vertex 사이 edge weight 합이 최소인 path.

- shortest path(edge 수 최소)와 다름 — weight 합 최소.
- 예: A→D 직접(weight 10)보다 A→B→C→D(weight 6)가 shortest weighted path.
- trust는 거리와 역상관 — 짧으면서 높은 trust를 동시에 찾는 bounded minimum 최적화 문제.

---

## 2. Search 최적화 & Supernode

> graph search 알고리즘은 path tree를 유지하며 heuristic으로 edge 추가 여부를 결정한다.

- **Lowest cost**: 목적 vertex에 더 짧은 path가 이미 있으면 edge 제외.
- **Supernode avoidance**: degree가 임계 넘는 vertex 제외.
- **Global heuristic**: path 총 weight가 임계 넘으면 edge 제외.
- **Supernode**: 비정상적으로 degree 높은 vertex(Twitter 유명인 = Ashton Kutcher 문제). traversal 시 priority queue 폭발.

---

## 3. Edge Weight Normalization (3 step)

> trust를 shortest path로 풀 수 있게 scale을 [0,1]로 옮기고 log·×(-1)로 변환한다.

- **Step1 [0,1] shift**: [-10,10]→[0,1]. -10→0(불신), 1→0.5(중립), 10→1(완전신뢰). 0 평점 없으므로 제거.
- **Step2 shortest path화**: log 적용(곱셈→덧셈, trust는 confidence라 곱해야 함) + ×(-1)(최대→최소). 0.30103이 trust/distrust 경계.
- **Step3 infinity 처리**: (-1)*log(0)=∞ → **100**으로 설정(길이 101 path여야 능가 → 사실상 배제).

---

## 4. Graph 갱신 & normalized weight 탐색

> norm_trust를 clustering key(Asc)로 추가해 disk에서 증가순 정렬.

```
schema.edgeLabel("rated").from("Address").to("Address").
  clusterBy("norm_trust", Double, Asc).property("datetime",Text).create()
```

- 변환 공식: 거리 d → trust scale = 10^(-d). 0이면 trust 1, 0.30103이 0.5 경계.
- 15 shortest path(length순)를 norm_trust로 정렬 → length 3 path(0.2899)가 length 2 path(0.32583)보다 trusted. 단 이는 shortest **weighted** path 아님.

---

## 5. Shortest Weighted Path 쿼리 — step 1~2

> repeat().until() 다음 order()는 barrier → 모든 path를 찾게 됨. minDist map으로 최적화 준비.

- **Step1**: order/limit 순서 swap + limit(1) → shortest weighted path. 단 order()가 barrier라 all-paths 처리(비효율).
- **Step2**: `group("minDist").by().by(sack().min())` → vertex별 최소 거리 lookup table 생성.

---

## 6. Lowest Cost 최적화 — filter + project().where()

> minDist map으로 현재 vertex까지 최소 거리와 traverser sack을 비교, 같으면 shortest path라 통과.

```
filter(project("a","b").
  by(select("minDist").select(select("visited"))).  // a = 최소거리
  by(sack()).                                        // b = 현재 거리
  where("a",eq("b")))                                // a==b면 생존
```

- 이 최적화부터 timeout 없이 반환 — 처리 path 수 감소 시작.

---

## 7. Supernode 회피 & Global Heuristic — and/sideEffect

> outgoing degree 100+ 제외, sack 1.0+ 제외를 and()로 묶는다.

```
and(<lowest cost test>,
  filter(sideEffect(outE("rated").count().is(gt(100)))),  // supernode 제외
  filter(sack().is(lt(1.0))))                              // global heuristic
```

- **and(t1,t2,...)**: 모든 traversal boolean AND. 셋 다 true여야 통과.
- **sideEffect(traversal)**: traverser 상태(위치) 변경 없이 부수 계산. degree 세려면 edge로 이동해야 하므로 sideEffect로 감싸 위치 보존.
- **filter()/where()**: boolean 평가로 false면 traverser 제거.

---

## 8. 결과 해석

> 최종 shortest weighted path는 length 7(0.1288) → 10^(-0.1288)=0.7434, trust로 결론.

- 단순 shortest(length)보다 더 길지만 trust 높은 path 발견.
- 0.1288 → trust score 0.7434 → 1337로부터 bitcoin 수령 신뢰 가능.

---

## Summary (핵심 정리)
- shortest weighted path = edge weight 합 최소 path (edge 수 최소와 다름).
- 최적화: lowest cost(더 짧은 path 있으면 제외), supernode avoidance(degree 임계), global heuristic(weight 임계). A* 등이 이를 적용.
- normalization 3단계: [0,1] shift → log(곱셈→덧셈)·×(-1)(최대→최소) → infinity는 100. trust scale 복원 = 10^(-d).
- Gremlin 구현: group("minDist")로 최소거리 추적, filter+project().where()로 lowest cost, and()+sideEffect()로 supernode·heuristic 필터.
- sideEffect는 위치 변경 없이 부수계산(degree count) — supernode 판별에 필수.
