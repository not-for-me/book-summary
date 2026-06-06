# 05. Query Analysis and Tuning

## 챕터 개요 (3줄 요약)

- Cypher 쿼리가 파싱→계획(planner)→실행(runtime)되는 파이프라인과, 패턴 앵커(anchor)·선택성(selectivity)·카디널리티(cardinality) 같은 성능의 핵심 개념을 설명한다.
- `EXPLAIN`/`PROFILE`로 실행 계획을 분석해 Rows·DB Hits·페이지 캐시(page cache)를 읽고, 카테시안 곱(Cartesian product)·이거(eager) 연산자·정렬 같은 비용 요인을 진단·튜닝한다.
- 인덱싱 가이드라인, 속성 접근 지연, 노드 차수(degree) 최적화, 플래너 힌트, 런타임 선택, 파라미터화, 쿼리 시간 측정까지 실전 튜닝 기법을 다룬다.

---

## 1. Query Execution and Pattern Anchors

> Cypher 쿼리는 추상 구문 트리(AST)로 파싱된 뒤, 통계·인덱스·제약을 활용하는 플래너가 실행 계획을 만들고 런타임이 이를 실행한다.

- 처리 파이프라인: 파싱 → 정규화/의미 분석 → 플래너(통계 기반 최적화) → 런타임 실행 → 결과.
- 플래너는 패턴에서 앵커(anchor, 진입점이 되는 노드/관계)를 식별하고, 페이지 캐시에 없으면 메모리에 로드한 뒤 거기서 확장(expand)하며 순회한다.
- 확장 도중 조건(predicate)·집계(aggregation)·변환을 평가한다.
- 좋은 앵커 선택이 쿼리 속도를 좌우하므로, 플래너가 선택적인(selective) 앵커를 빨리 찾도록 돕는 것이 튜닝의 목표다.

---

## 2. Query Profiling and Cardinality

> `PROFILE`은 실행 계획의 각 연산자가 처리한 행 수와 저장 엔진 작업량을 보여주어 병목을 드러낸다.

- Rows: 각 연산자가 통과시킨 행 수 — 카디널리티가 폭발하는 지점을 찾는다.
- 데이터베이스 히트(DB Hits): 저장 엔진에 데이터 저장/조회를 요청한 추상 작업 단위로, IO를 유발할 수 있다(노드 조회, 라벨/속성 읽기, 인덱스 시크 등).
- 페이지 캐시 적중/실패(page cache hits/misses): 데이터가 메모리에 있었는지 디스크에서 읽었는지를 나타낸다.
- 행 카디널리티(row cardinality): Cypher는 스트리밍("lazy") 방식이라 대부분의 연산자가 행을 즉시 다음으로 흘려보내지만, 이거(eager)·정렬·집계는 예외다.

---

## 3. Selectivity, Disconnected Patterns, and Indexing

> 모든 쿼리는 앵커에서 시작하므로, 선택성을 높이고 불필요한 카테시안 곱을 피하며 인덱스를 신중히 설계해야 한다.

- 분리된 패턴 매칭(disconnected patterns): 연결되지 않은 패턴(예: `MATCH (a:Artist),(t:Track)`)은 카테시안 곱(CartesianProduct, 교차 조인)을 만들어 매우 메모리 집약적이므로 `EXPLAIN`만 사용해 확인해야 한다.
- 앵커 선택성 높이기: 조건이 없으면 그래프 통계(라벨별 노드 수, 타입별 관계 수)를 참조하도록 충분한 정보를 제공한다.
- 중복 필터 제거 및 조건이 있는 쿼리의 앵커 개선으로 플래너가 더 선택적인 진입점을 고르게 한다.
- 인덱싱 가이드라인: 모든 것을 인덱싱하면 저장 공간이 (최악의 경우) 두 배가 되고 쓰기 처리량이 떨어지므로, 선택적으로 인덱스를 만든다.

---

## 4. Properties, Node Degrees, Eager, Sorting

> 속성 접근은 늦출수록 좋고, 관계 차수 계산이나 이거 연산자는 메모리/IO 비용을 키울 수 있다.

- 속성 접근(accessing properties): 노드/관계의 속성 읽기를 가능한 한 늦게(보통 `RETURN` 시점) 미뤄 불필요한 비용을 줄인다.
- 노드 차수(node degrees): "n개 이상의 플레이리스트에 있는 트랙 찾기"처럼 차수가 중요한 경우, 관계 카운트 최적화(예: 차수 저장소 활용)를 고려한다.
- 이거 연산자(Don't Be Eager!): 정렬·집계 등은 모든 행을 모은 뒤 진행하므로 힙에 데이터를 쌓아 메모리를 많이 쓴다 — 가능하면 회피한다.
- 정렬(sorting): `ORDER BY`의 `Sort` 연산자는 이거 연산자로, 모든 행을 메모리에 올려 정렬하므로 메모리 사용이 증가한다.

```
Cypher pipeline:
  query string -> parse(AST) -> normalize -> PLANNER(stats,index) -> plan
                                                                      |
  RUNTIME executes plan: anchor -> expand -> predicates -> aggregate -> RETURN
```

---

## 5. Planner Hints, Runtimes, Parameters, Monitoring

> 드물게 플래너보다 더 잘 알 때는 힌트를 주고, 런타임 특성과 파라미터화를 이해하면 추가 성능을 얻는다.

- 플래너 힌트(I Want to Break Free): 인덱스가 여럿일 때 등 드문 경우 플래너에 힌트를 제공해 마지막 밀리초까지 짜낸다.
- Cypher 런타임: 파이프라인(pipelined)이 Enterprise 기본 런타임으로, 연산자가 행 배치를 버퍼로 주고받아 효율적이다(다른 런타임도 존재).
- 쿼리 파라미터화(parameterizing queries): 파라미터를 쓰면 캐시된 실행 계획을 재사용해 반복 파싱/계획을 피한다.
- 쿼리 시간 모니터링/측정: 프로덕션 투입 전 실행 시간을 측정하며, 테스트 그래프가 프로덕션 그래프의 형태와 통계를 닮아야 정확하다.

---

## Summary (핵심 정리)

- 쿼리 튜닝의 핵심은 플래너가 선택적인 앵커를 빨리 찾도록 돕고, `PROFILE`로 Rows·DB Hits·페이지 캐시를 읽어 병목을 진단하는 것이다.
- 카테시안 곱·이거 연산자·과도한 속성 접근·불필요한 인덱스를 피하고, 파라미터화로 실행 계획을 재사용한다.
- 드물게 플래너 힌트와 런타임 선택으로 추가 성능을 얻되, 프로덕션과 유사한 테스트 그래프에서 측정하는 것이 중요하다.
