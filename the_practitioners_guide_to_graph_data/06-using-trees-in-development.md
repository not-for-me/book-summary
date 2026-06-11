# 06. Using Trees in Development

## 챕터 개요 (3줄 요약)
- hierarchical(nested) 데이터를 graph의 tree로 다루며, BOM/version control/조직도 등 실세계 예제와 tree 관련 용어(root, leaf, depth, walk, path, cycle)를 정리한다.
- Edge Energy의 self-organizing sensor 통신망을 예제로 leaf→root, root→leaf 두 방향 traversal을 Gremlin으로 구현한다.
- repeat()/until(), simplePath(), path()/as()/by() 패턴을 익히고, root→leaf 재귀 walk가 cycle·depth 때문에 깨지는 문제를 노출(Ch7 복선).

---

## 1. Hierarchical 데이터의 3가지 실세계 예제

> nested 의존 구조는 graph의 tree로 자연스럽게 표현된다.

- **BOM(Bill of Materials)**: 제품 구성의 중첩 의존성(부품·수량). 예: 737의 나사 총 개수.
- **Version control(Git)**: working directory/index/head 세 tree, fork 있는 의존 chain.
- **Self-organizing network**: family tree, 조직도(manager-employee = parent-child).
- graph는 자연스러운 표현 → 코드 단순화(예: HBase 150줄 → Gremlin 20줄), 생산성↑.

---

## 2. Tree 용어 — root, leaf, parent, child, forest

> tree = cycle 없는 connected graph.

- **Tree**: cycle 없는 연결 graph. 여러 tree = **forest**.
- **Parent/Child**: 계층상 한 단계 위/아래 vertex.
- **Root**: 최상위 parent(의존 chain 시작). **Leaf**: 마지막 child(degree 1).

---

## 3. Depth, Walk, Path, Cycle

> 계층 데이터는 neighborhood·depth·path 세 방식으로 참조된다.

- **Depth**: vertex에서 root까지의 거리.
- **Walk**: vertex·edge 방문 sequence (반복 허용).
- **Path**: vertex·edge 방문 sequence (반복 불가).
- **Cycle**: 시작·끝 vertex가 같은 path. (loop = 한 vertex의 자기 edge, cycle과 다름)

---

## 4. Sensor 데이터 이해 (Edge Energy)

> sensor가 reading을 인접 sensor 또는 tower로 전달하는 동적·계층 통신망.

- sensor(asterisk)→인접 sensor/tower로 send, 최종적으로 tower→모니터링 시스템 도달.
- bottom-up: 한 sensor에서 tower까지 가능한 다양한 path(거리 1~6, 1000+개).
- top-down: tower에서 depth별 도달 가능 sensor.
- 실세계 hierarchy는 perfect tree가 아님 — **cycle 존재**(loop는 없음).

---

## 5. Schema & 로딩

> Sensor/Tower vertex(geo Point 포함), self-referencing send edge.

```
schema.vertexLabel("Sensor").partitionBy("sensor_name",Text)
  .property("coordinates",Point)...create();
schema.edgeLabel("send").from("Sensor").to("Sensor").create()  // self-referencing
schema.edgeLabel("send").from("Sensor").to("Tower").create()
```

- self-referencing edge label은 loop와 다름(schema 개념 vs 데이터 개념).
- dsbulk 로딩 시 self-ref edge는 out_/in_ prefix 자동 생성. timestep 컬럼은 무시(Ch7에서 사용).

---

## 6. Leaf→Root 쿼리 — neighborhood & coalesce

> sensor에서 send edge를 walk해 도달 vertex를 탐색하고 project/coalesce로 결과 shaping.

```
dev.V(sensor).out("send").
  project("Label","Name").
  by(label).
  by(coalesce(values("tower_name","sensor_name")))  // tower면 tower_name, 아니면 sensor_name
```

- neighborhood를 늘릴수록 cycle 발견(예: 1035508→1307588→1035508).

---

## 7. repeat()/until() + simplePath() + path()

> 임의 depth로 tower까지 재귀 walk하되 cycle은 simplePath()로 제거한다.

```
dev.V(sensor).
  until(hasLabel("Tower")).         // do/while (until이 repeat 뒤면 do/while)
  repeat(out("send").simplePath()). // cycle 제거
  path().by(coalesce(values("tower_name","sensor_name")))
```

- **simplePath()**: path에 반복 객체 있으면 traverser 필터(cycle 제거). 없으면 무한 루프.
- **path()**: traverser의 전체 이력 반환(빵부스러기). labels/objects 두 key.

---

## 8. path()의 as()와 by()

> as()로 path 객체에 변수명(label)을 부여하고, by()로 각 객체를 round-robin으로 가공한다.

- **as("start"/"visited"/"tower")**: path의 labels key를 채움. labels↔objects는 1:1 매핑.
- **by() round-robin**: 첫 by는 1·3·5번째 객체, 두 번째 by는 2·4번째 객체에 순환 적용.
- path data structure의 labels는 vertex/edge label과 다름(주의).

---

## 9. Root→Leaf 쿼리와 실패 (Ch7 복선)

> tower에서 모든 sensor로 재귀 walk는 cycle·depth 폭발로 timeout된다.

```
dev.V(tower).repeat(__.in("send").simplePath())  // 30s timeout 에러!
// in()은 Groovy 예약어 → Anonymous traversal __. 필요
```

- group().by().by(inE().count()) + cap()(barrier) + order(local)로 degree 분포 조사(Georgetown=7).
- until(hasLabel("Sensor"))는 첫 neighborhood만 → 잘못. 조건 제거하면 timeout.
- **depth limiting**: `repeat(...).times(3)`로 깊이 제한 → but 일부 sensor 누락. 전체는 time(Ch7) 필요.

---

## Summary (핵심 정리)
- hierarchical 데이터(BOM/Git/조직도)는 graph의 tree(cycle 없는 connected graph)로 표현, forest=여러 tree.
- 용어: root/leaf, parent/child, depth(root까지 거리), walk(반복O)/path(반복X)/cycle(시작=끝), loop≠cycle.
- Edge Energy sensor망: bottom-up(sensor→tower path 다수) & top-down(tower→sensor). 실세계는 cycle 존재.
- 핵심 Gremlin: repeat()/until()(재귀), simplePath()(cycle 제거), path()+as()(label)+by()(round-robin shaping), Anonymous traversal __.
- root→leaf 전체 재귀는 cycle·depth 폭발로 timeout → times(x) depth limiting은 부분해, 완전한 해법은 time 기반(Ch7).
