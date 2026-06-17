# 03. Data Models and Query Languages

> Keywords: relational vs document model, impedance mismatch/ORM, normalization vs denormalization, one-to-many/many-to-many, schema-on-read vs schema-on-write, data locality, star/snowflake/OBT schema, declarative query language, property graph/Cypher, triple store/RDF/SPARQL, Datalog, GraphQL, event sourcing/CQRS, DataFrame/matrix

## 챕터 개요 (3줄 요약)
- data model은 소프트웨어 작성 방식과 문제를 바라보는 사고방식 자체를 규정하는 가장 중요한 선택이다.
- relational·document·graph·event sourcing·DataFrame을 relationship 유형(one-to-many/many-to-many) 관점에서 비교하고, 각 model의 query language를 살펴본다.
- 하나의 model로 다른 model을 흉내낼 수 있으나 어색하다 — 데이터의 관계 구조에 맞는 model 선택이 핵심이며, 최근 model 간 수렴(convergence) 추세.

---

## 1. Relational Versus Document Models
> document는 self-contained JSON tree(one-to-many)에 강하고, relational은 join·many-to-one/many-to-many에 강하다.

### Object-Relational Mismatch & ORM
- `impedance mismatch`: OOP 객체와 relational table 간 번역 계층 필요. ORM(ActiveRecord, Hibernate)이 boilerplate 줄임.
- ORM 문제: 두 model 모두 신경써야 함, OLTP에 국한, 비효율 query 유발(특히 `N+1 query problem`).
- ORM 장점: 단순·반복 케이스 boilerplate 감소, query 캐싱, schema migration 보조.

### One-to-many → 문서가 자연스러움
- LinkedIn 이력서 예: positions/education/contact는 one-to-many(또는 one-to-few) tree → JSON 한 문서로 locality↑, multiway join 회피.
- 단 관련 항목이 매우 많으면(celebrity 댓글 수천 개) embed는 부적합 → relational이 나음.

### Normalization vs Denormalization
- `normalized`: 사람이 읽는 정보를 한 곳에만 저장, 나머지는 ID 참조(ID는 불변). write 빠름, read는 join 필요(느림).
- `denormalized`: 정보를 record마다 복제. read 빠름(join↓), write 비싸고(복제 갱신) inconsistency 위험. denormalization은 derived data의 일종.
- 일반 원칙: OLTP는 normalized 선호, analytics는 denormalized 선호. 대규모에선 join 비용이 문제될 수 있음.
- X timeline 사례: materialized timeline은 post의 text가 아니라 post ID/sender ID만 저장 → read 시 `hydrating`(ID→실데이터 join, app code). 자주 바뀌는 like/profile은 denormalize하지 않음. join이 곧 확장성의 적은 아님(잘 병렬화됨).

```
normalized:   write fast, read slow (joins),  single source
denormalized: read fast,  write slow (updates), duplicated copies
```

### Many-to-one / Many-to-many
- relational: associative table(join table)로 표현. 양방향 query는 secondary index로.
- document: many-to-many는 self-contained 문서에 안 맞음 → 다른 문서를 ID로 참조(normalized 권장).

### Stars and Snowflakes (analytics schema)
- `star schema`: 중앙 `fact table`(이벤트 1행=1구매 등) + 주변 `dimension table`(who/what/where/when). fact는 dimension에 FK.
- `snowflake schema`: dimension을 subdimension으로 더 정규화(별→눈송이). star가 분석가에게 더 단순해 선호.
- `OBT`(one big table): dimension을 fact에 fold(join 미리 계산). 저장↑이지만 query↑. analytics는 데이터 불변이라 denormalization 부담 적음.

### When to Use Which / Schema flexibility
- document 장점: schema flexibility, locality 성능, object model 근접. 단점: 중첩 항목 직접 참조 불가, reorderable list엔 강함.
- `schema-on-read`(구조는 읽을 때 해석, 동적 타입 유사) vs `schema-on-write`(DB가 write 시 강제, 정적 타입 유사).
- format 변경 시: document는 app code에서 구버전 처리, relational은 ALTER+UPDATE migration(대형 테이블은 느림).
- heterogeneous 데이터엔 schema-on-read 유리, 동질 데이터엔 schema가 문서화·강제에 유용.
- `data locality`: 문서는 연속 저장이라 전체 접근 시 유리하나, 일부만 필요하면 낭비. Spanner의 interleaved table, Bigtable의 column family도 locality 제공.
- 수렴(convergence): relational이 JSON 지원, document가 join·secondary index·선언 query 추가. relational-document hybrid가 강력.

---

## 2. Graph-Like Data Models
> many-to-many가 흔하고 다중 hop traversal이 필요하면 graph가 자연스럽다.

### 기본 개념
- graph = `vertices`(node/entity) + `edges`(relationship/arc). social graph, web graph, road network 등.
- 표현: adjacency list(traversal에 강함) vs adjacency matrix(ML에 강함).
- 이종(heterogeneous) 객체를 한 graph에 일관되게 저장 가능(Facebook 단일 graph, search engine `knowledge graph`).
- 두 모델: `property graph`(Neo4j, Memgraph, KùzuDB) / `triple store`(Datomic, AllegroGraph). 표현력 유사.

### Property Graph & Cypher
- property graph: 각 vertex(id, label, 양방향 edges, properties) / 각 edge(id, tail, head, label, properties).
- relational로 표현 시 vertices·edges 두 테이블(+tail/head index)로 가능. 모든 vertex가 임의 vertex와 연결 가능, 양방향 traversal, 다양한 label로 이종 저장.
- `Cypher`(Neo4j 발, openCypher 표준): `(idaho)-[:WITHIN]->(usa)` 화살표 표기. `:WITHIN*0..`는 'WITHIN edge 0회 이상'(정규식 * 유사)으로 가변 길이 traversal.

### Graph Queries in SQL
- graph를 relational에 넣고 SQL로 query 가능하나, 가변 길이 traversal엔 `WITH RECURSIVE`(recursive CTE) 필요 — 4줄 Cypher가 31줄 SQL이 됨(model/언어 선택의 중요성).
- 2024년 `GQL` ISO 표준(Cypher 기반) 발표.

### Triple Stores & SPARQL / RDF
- triple: (subject, predicate, object). object가 primitive면 property, 다른 vertex면 edge. property graph와 거의 동치.
- Turtle/N3 표기, `RDF`(Resource Description Framework, Semantic Web 유래). predicate가 URI인 건 데이터 통합 시 충돌 회피 목적.
- `SPARQL`: RDF용 query language(Cypher의 조상). `?person :bornIn / :within* ?location` 식 패턴 매칭.

### Datalog
- 1980년대 학술 기원, Prolog의 subset. relational 기반이지만 recursive query에 강함(Datomic, LogicBlox, CozoDB).
- facts(=행) 위에 rule(`:-`)로 virtual table(view)을 점진 정의, rule이 자기 자신 호출 가능 → graph traversal. 함수형 사고 유사.

### GraphQL
- 의도적으로 제한적. OLTP용 — client(mobile/web)가 UI 렌더링에 필요한 필드 구조의 JSON을 요청.
- untrusted source라서 recursive query·임의 검색 불허(DoS 방지). 응답은 query 구조를 그대로 미러링(필요한 필드만).
- 서버는 normalized로 저장하고 join 수행하나, schema에 선언된 join만 요청 가능. 어떤 DB 위에도 구현 가능('graph' 이름과 무관).

---

## 3. Event Sourcing and CQRS
> write 최적화 표현(immutable event log)에서 read 최적화 표현(materialized view)을 파생.

- `event sourcing`: 모든 상태 변화를 immutable event로 append(절대 수정/삭제 안 함). 과거형으로 명명('seats were booked').
- `CQRS`(command query responsibility segregation): write 모델과 read 모델(materialized view/projection) 분리. command 검증 후 valid event만 log에 추가.
- 장점: 의도 전달 명확, view를 같은 event로 재현 가능(버그 시 재계산), 다수 view 최적화, 새 view 쉽게 추가, 오류는 보정 event로 되돌림(irreversibility↓), audit log, 높은 write throughput(sequential).
- 단점: 외부 정보(환율 등)는 결정성 위해 event에 포함하거나 historical query 필요, GDPR 삭제 vs immutability(`crypto-shredding`), 부작용(이메일 재발송) 주의.
- 구현: EventStoreDB, MartenDB, Axon, Kafka+stream processor(Ch12). 필수 조건: 모든 view가 동일 순서로 event 처리.

```
Command -> validate -> Event Log (immutable, append-only)
                          |--> Materialized View A (bookings status)
                          |--> Materialized View B (dashboard charts)
                          |--> Materialized View C (badge printer files)
```

---

## 4. DataFrames, Matrices, and Arrays
> analytics/scientific 맥락의 model. relational을 넘어 다수 column·matrix로 일반화, ML의 기반.

- `DataFrame`(R, Pandas, Spark, Dask): table 유사하나 선언 query 대신 명령 series로 점진 'wrangling'. relational 연산(filter/group/merge=join) + 그 이상.
- relational → matrix 변환이 흔한 용도(예: user×movie 평점 sparse matrix). 비수치 데이터는 `one-hot encoding`, 날짜 scaling 등으로 수치화.
- matrix화하면 linear algebra(ML 알고리즘 기반) 적용 가능. `array database`(TileDB)는 대형 다차원 배열(geospatial, 의료영상, 천문) 특화. 금융 time-series에도 활용.

---

## Summary (핵심 정리)
- relational model(반세기+)은 여전히 핵심, 특히 data warehouse의 star/snowflake+SQL. document는 self-contained tree·관계 희소한 경우, graph는 '모든 것이 연결'되고 다중 hop traversal(Cypher/SPARQL/Datalog recursive query)이 필요한 경우.
- DataFrame은 relational을 다수 column·다차원 배열로 확장해 DB와 ML을 잇는다. event sourcing은 immutable event log(write 최적)에서 CQRS로 read 최적 view를 파생.
- 공통점: 비관계형 model은 보통 schema를 강제하지 않음 — schema가 명시(write)냐 암시(read)냐의 차이. model 간 경계는 점점 흐려지는 convergence 추세.
- 다음 연결: Ch4에서 이 data model들을 구현하는 storage engine의 trade-off를 다룸.