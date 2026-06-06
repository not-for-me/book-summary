# 3. Graph Databases

## 챕터 개요 (3줄 요약)
- 그래프 DB(이 책은 Neo4j 사용)와 선언적 패턴매칭 쿼리 언어 Cypher(현재 ISO GQL로 표준화 중)가 지식그래프를 실무적으로 구현 가능하게 만든 핵심 기술이다.
- Cypher는 ASCII 아트처럼 그래프 구조를 시각적으로 그대로 표현(노드=(), 관계=-[]->)하며, CREATE/MERGE/MATCH/WHERE로 데이터 생성·정합성·질의를 수행한다.
- Neo4j의 핵심 성능 원리는 "index-free adjacency"로, 쿼리 지연시간이 그래프 전체 크기가 아니라 "탐색한 양"에 비례한다(O(1) 트래버설).

## The Cypher Query Language
> Cypher는 사람이 다이어그램을 그리고 읽듯이 직관적이도록 설계된 선언적·시각적 패턴매칭 언어이다.

- 노드는 `()`, 레이블은 `:Person`, 관계는 `-[:LIVES_IN]->`, 속성은 `{name:'Rosa'}`로 표현 — "그린 것이 곧 저장되는 것".
- "도메인에 대해 아는 것이 있으면 그래프에 넣어라" — 레이블은 역할 그룹화 + 인덱스 등록으로 충실도와 성능을 동시에 높인다.

## Creating Data / Avoiding Duplicates
> CREATE는 항상 새 레코드를 만들고, MERGE는 "패턴 전체가 없을 때만" 생성하는 MATCH+CREATE 혼합 의미를 가진다.

- MERGE는 부분 매칭/부분 생성을 절대 하지 않음 → 전체 패턴을 한 번에 MERGE하면 중복(예: London 노드 중복) 발생.
- 해결: 단계 분해 + 제약조건. `CREATE CONSTRAINT ... REQUIRE (p.country, p.city) IS NODE KEY`로 복합 유니크 키를 걸어 중복을 안전하게 거부.
- DELETE(dangling 시 중단), DETACH DELETE(노드+관계 삭제), SET/REMOVE(속성·레이블 추가/제거).

```
// 제약조건 후 트랜잭션 단위 MERGE 패턴
MERGE (london:Place {city:'London', country:'UK'})
MERGE (fred:Person {name:'Fred'})
MERGE (fred)-[:LIVES_IN]->(london)
```

## Graph Local vs Global Queries
> 특정 노드에 묶인 질의(graph local)와 그래프 전체를 처리하는 집계 질의(graph global)를 구분한다.

- Local 예: "베를린에 사는 사람?" — `MATCH (p:Person)-[:LIVES_IN]->(:Place {city:'Berlin'})`.
- 가변 길이 경로 `-[:FRIEND*2..2]->`로 friends-of-friends 탐색; 자기 자신 제외는 `WHERE rosa <> fof` 술어로 처리.
- 관계형 DB와 달리 깊이 증가에 join-bomb 없음 — 단일 관계 트래버설 비용은 수백만분의 1초 수준.
- Global 예: "가장 인기있는 도시?" — `count()`, `avg/max/min/sum`, `ORDER BY`, `SKIP/LIMIT`로 대규모 그래프에서 압축된 집계 결과 산출.

## Functions/Procedures & Tools
> CALL 구문으로 프로시저(APOC 라이브러리 등)와 함수를 호출하고, EXPLAIN/PROFILE로 쿼리 성능을 진단한다.

- `db.schema.visualization()`로 스키마 확인, APOC는 반복 작업을 줄이는 풍부한 유틸리티 제공.
- EXPLAIN(실행 안 하고 계획 시각화), PROFILE(실제 실행 계측) — "DB hits"와 시간 비용을 줄이고, 스캔이 많으면 인덱스 추가를 검토.

## Neo4j Internals
> "엔지니어일 필요는 없지만 mechanical sympathy(기계에 대한 공감)는 필요하다" — 내부를 알면 결을 거스르지 않는 설계가 가능하다.

- Index-free adjacency: 노드/관계를 고정 길이 레코드로 분리 저장, 관계 자체가 로컬 "인덱스" 역할 → O(1) 트래버설(포인터 체이싱).
- 속성 저장은 O(N) 읽기(리스트) → 비싸므로 "구조를 먼저 트래버스하고 속성은 나중에 조회"하는 것이 성능 핵심. RAM 비율을 충분히 둬라(가능하면 전체 그래프를 RAM에).
- ACID 트랜잭션: write-ahead log로 내결함성, 분산 환경에서 Raft 알고리즘 + causal barrier(최소 자기 쓰기 보장).

> [모델링 관점 - 주식시장 도메인 적용]
> 주식시장 그래프는 본질적으로 "관계가 깊고 빈번한 트래버설" 워크로드다(예: 공급망 N홉 전파, 지분 보유 체인, 상관관계 클러스터). index-free adjacency 덕분에 "이 기업의 3차 공급사까지의 리스크 전파"를 그래프 크기와 무관하게 빠르게 질의할 수 있다는 점이 RDB 대비 결정적 이점이다. 모델링 시: (1) 자주 트래버스하는 경로는 구조(관계)로, 필터링 속성은 최소화. (2) 종목/기업의 식별자(ticker, ISIN)에 NODE KEY 제약을 걸어 중복 종목 노드를 방지(데이터 통합 시 필수). (3) 시계열 가격·거래량 같은 대량 속성은 트래버설 경로에서 분리해 성능 저하를 막아라.

## Summary (핵심 정리)
- Cypher는 ASCII 아트 기반 패턴매칭으로 그래프를 직관적으로 생성·질의하며, MERGE+제약조건이 데이터 정합성(중복 방지)의 핵심 도구다.
- Neo4j의 index-free adjacency는 "쿼리 비용이 탐색량에 비례"하게 만들어 깊은 관계 탐색에 강하며, 속성 접근은 비싸므로 구조 우선 탐색이 성능 원칙이다.
- 주식시장처럼 깊은 관계 전파를 다루는 도메인에서는 식별자 제약·구조/속성 분리·대량 시계열의 외부화가 실무 모델링의 핵심 고려사항이다.
