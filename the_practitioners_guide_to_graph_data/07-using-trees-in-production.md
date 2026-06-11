# 07. Using Trees in Production

## 챕터 개요 (3줄 요약)
- edge에 timestep을 모델링해 valid path(시간 순서 정합) 개념을 도입하고, branching factor의 지수적 폭발 문제를 설명한다.
- edge를 time으로 cluster하고 materialized view를 추가한 production schema로 처리 데이터를 대폭 줄인다.
- loops()/sack()와 where().by() 필터로 valid tree만 walk하고, tower 장애 시나리오(at-risk sensor)를 해결한다.

---

## 1. Time on Edges & Valid Path (bottom-up)

> message가 timestep n에 전달되면 다음 sensor는 n+1에 전달 — 시간이 1씩 증가해야 valid.

- vertex 수는 고정, relationship(edge)이 시간에 따라 증가 → timestep property로 모델링.
- bottom-up(sensor→tower): 시간이 1씩 **증가**해야 valid. 늦거나 이른 전달은 invalid.
- Ch6의 7개 path 중 2개는 시간 정합 안 되어 실제로 불가능.

---

## 2. Valid Path (top-down) & Rule of Thumb #11

> top-down(tower→sensor): 시간이 1씩 **감소**해야 valid.

- tower에서 sensor로 reverse walk = 시간을 거꾸로(감소) 추적.
- **#11**: "올라갈 땐 time↑, 내려갈 땐 time↓. 아니면 invalid path → 필터링."

---

## 3. Branching Factor

> BF = vertex당 평균 edge 수. depth가 깊어지면 traverser 수가 지수적으로 폭발한다.

- **Branching factor**: 임의 vertex의 기대(평균) edge 수. WestLake=7.
- traverser ≈ thread. 총 처리량 ≈ BF^depth 누적 (BF=3, depth4면 1+3+9+27+81=121).
- Ch6의 timeout 원인 = root→leaf 전체 재귀 시 BF 폭발.

---

## 4. Branching Factor 완화 & Rule of Thumb #12

> edge를 disk에서 cluster해 쿼리 시 정렬·필터로 탐색 범위를 줄인다.

- **#12**: edge를 cluster해 쿼리에서 sort/필터 → BF 영향 완화.
- valid path만 고려하면 처리 데이터가 크게 감소.

---

## 5. Production Schema — clusterBy(time) + materialized view

> edge를 timestep desc로 cluster하고 양방향 walk 위해 inverse MV를 추가한다.

```
schema.edgeLabel("send").from("Sensor").to("Sensor").
  clusterBy("timestep", Int, Desc).create()
schema.edgeLabel("send")...materializedView("sensor_sensor_inv").inverse().create()
```

- **Bonus RoT**: production edge는 가장 자주 walk하는 방향, MV는 덜 쓰는 방향으로.
- 데이터·로딩은 Ch6과 동일하나 이번엔 timestep이 edge에 실림.

---

## 6. Leaf→Root: loops()로 valid tree 필터

> loops()(0부터 1씩 증가)와 edge timestep을 where().by()로 비교해 valid path만 통과시킨다.

```
g.V(sensor).as("start").
  until(hasLabel("Tower")).
  repeat(outE("send").as("send_edge").
    where(eq("send_edge")).by(loops()).by("timestep").  // loops == timestep?
    inV().as("visited")).
  path()...
```

- path 구조 [Start, Edge, Vertex, ...], by() round-robin: 짝수=vertex, 홀수=edge.

---

## 7. 흔한 실수 — has() overloading & where().by() 원리

> has("timestep", loops())는 has(True)처럼 항상 통과한다 — 잘못된 필터.

- **has(key, traversal)**: loops()가 값을 반환하기만 하면 통과 → 비교가 아님(overloaded).
- **where(eq("send_edge")).by(loops()).by("timestep")**: 같은 edge에서 두 by()가 다른 값(loops vs timestep) 내면 false → 제거. 이것이 올바른 비교.

---

## 8. Root→Leaf: sack()으로 감소 카운터

> loops()는 증가만 가능 → top-down은 감소가 필요하므로 sack()(traverser별 backpack)을 사용한다.

```
g.withSack(start).V(tower).as("start").
  repeat(inE("send").as("send_edge").
    where(eq("send_edge")).by(sack()).by("timestep").
    sack(minus).by(constant(1)).        // sack 1씩 감소
    outV().as("visited").simplePath()).  // cycle 제거
  times(start+1).path()...
```

- **sack()/withSack()**: traverser 로컬 데이터 구조 초기화·읽기·쓰기. sack(minus)로 감소.
- simplePath()로 cycle 제거 필수.

---

## 9. Tower 장애 시나리오 해결

> 두 쿼리를 method로 감싸 at-risk sensor(Georgetown만 통신한 sensor)를 식별한다.

- getSensorsFromTower(): 시간 window를 돌며 Georgetown과 통신한 모든 sensor 수집(atRiskSensors).
- getTowersFromSensor(): 각 sensor가 통신한 다른 tower 조회(otherTowers map).
- Georgetown만 연결한 sensor = at-risk(orange), 다른 tower도 연결 = 안전(green).
- 결과로 Edge Energy에 proactive한 장애 대응 인사이트 제공.

---

## Summary (핵심 정리)
- edge timestep으로 valid path 정의: bottom-up은 time↑, top-down은 time↓ (RoT #11).
- branching factor(BF^depth)가 재귀 walk를 지수적으로 폭발시킴 → edge cluster로 완화(RoT #12).
- production schema: clusterBy(timestep, Desc) + inverse materialized view(양방향).
- valid tree 필터: 증가는 loops(), 감소는 sack() + where(eq).by(counter).by("timestep"). has(key, traversal) overloading 주의.
- 두 방향 쿼리를 method로 합쳐 tower 장애 시 at-risk sensor 식별 — 복잡한 문제를 graph로 분해 해결.
