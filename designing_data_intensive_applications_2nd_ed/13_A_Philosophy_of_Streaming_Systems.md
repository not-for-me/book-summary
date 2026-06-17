# 13. A Philosophy of Streaming Systems

> Keywords: data integration, deriving data vs distributed transaction, total order/causality, batch+stream(lambda/kappa), unbundling database, federated database/polystore, dataflow application, write path vs read path, materialized view/cache, stateful/offline client, end-to-end argument, exactly-once/idempotence/request ID, uniqueness constraint requires consensus, multishard processing, timeliness vs integrity, coordination-avoiding, compensating transaction, trust but verify/auditability

## 챕터 개요 (3줄 요약)
- 단일 도구로 모든 use case를 충족할 수 없으므로 여러 시스템을 조합한다 — 한 시스템을 system of record로 두고 나머지를 derived data로 비동기 파생하는 것이 핵심 철학.
- database의 기능(index·view·replication)을 unbundling해 log 기반 loosely coupled component로 조합하며, dataflow(write path/read path)로 application을 설계한다.
- correctness는 distributed transaction이 아니라 end-to-end(request ID idempotence)·비동기 constraint 검사로 달성 — timeliness와 integrity를 분리해 coordination을 회피한다.

---

## 1. Data Integration
> 한 곳에 먼저 write하고 나머지를 같은 순서로 derive — total order가 핵심.

- 문제마다 여러 해법(trade-off). 복잡한 app은 여러 도구 조합 필요(OLTP+search index+DW+cache+ML+notification).
- dataflow 명확히: 어디에 먼저 write? 무엇이 무엇에서 derive? CDC로 한 시스템을 통하면 일관성 보장. `dual write`는 순서 역전·부분 실패로 위험 → 단일 시스템이 total order 결정(state machine replication).
- derived data vs distributed transaction: 추상적으로 유사(atomic commit vs deterministic retry+idempotence). transaction은 read-your-writes 보장, derived는 비동기. XA의 fault tolerance·성능 한계 → log 기반 derived가 유망.
- total order 한계: 단일 leader throughput·다지역·microservice·offline client에서 순서 미정. `causality` 포착이 필요(logical timestamp, event ID 참조, conflict resolution — 단 외부 side effect엔 부족).

### Batch & Stream
- 둘 다 derived state 유지. batch는 deterministic·pure function(immutable input). stream은 fault-tolerant state 추가. 비동기가 fault 격리(distributed transaction은 실패 전파).
- reprocessing으로 application evolution(새 schema를 old와 병렬 운영, 점진 migration — 철도 dual gauge 비유, 가역적이라 안전).
- `lambda architecture`(문제 많아 사용 감소) → `kappa architecture`(batch+stream 통합: historical replay, exactly-once, event time windowing — Apache Beam).

---

## 2. Unbundling Databases
> database의 index 유지 기능을 분리해 log 기반으로 이종 시스템에 동기화.

- DB·batch/stream·OS는 본질적으로 같음(저장+처리). Unix(저수준 abstraction, pipe) vs relational(고수준, SQL/transaction) 철학 대립. 둘의 장점 결합 시도.
- `CREATE INDEX`는 snapshot scan+backlog 처리 = follower replica 설정·CDC bootstrap과 유사. 조직 전체 dataflow가 하나의 거대한 database(batch/stream은 trigger·materialized view 유지의 정교한 구현).
- 두 방향: `federated database`/polystore(read 통합 — Trino, PostgreSQL FDW), `unbundled database`(write 동기화 — CDC+event log+idempotent). log 기반 통합의 장점: loose coupling(시스템·인간 레벨 fault 격리·독립 개발).
- unbundling은 단일 DB 성능 경쟁이 아니라 breadth(넓은 workload). 단일 도구로 충분하면 그것을 써라.

### Designing Applications Around Dataflow
- spreadsheet(VisiCalc 1979)처럼 input 변경 시 derived 자동 갱신이 이상. application code = derivation function(secondary index, full-text, ML model, cache).
- state와 application code 분리('separation of Church and state'). DB는 mutable shared variable이나 변경 구독 불가(polling). stream processor가 ordered·fault-tolerant 갱신 제공.
- stream processor vs microservice: 단방향 async stream vs sync request/response. dataflow는 더 빠르고(로컬 DB query=환율 stream join) robust(network 호출 없음). 단 time-dependent join 주의.

### Observing Derived State
- `write path`(eager 사전계산) vs `read path`(lazy, 요청 시). derived dataset이 둘의 경계 = write/read 작업량 trade-off. index·cache·materialized view는 경계를 이동.
- stateful/offline client: on-device state = server state의 cache(pixel=model의 materialized view). server-sent events·WebSocket으로 state 변경을 client까지 push(write path를 end user까지 확장). offline은 consumer offset처럼 재연결.
- reads are events too: read를 event로 표현해 write와 같은 stream operator로(stream-table join). causal dependency·provenance 추적. multishard 복잡 query 분산 실행(Storm distributed RPC, fraud 점수).

---

## 3. Aiming for Correctness
> distributed transaction 없이 end-to-end·비동기로 integrity를 달성.

### End-to-End Argument
- serializable transaction도 app 버그(잘못된 write)는 못 막음 → immutable·append-only가 복구 유리.
- `exactly-once`: idempotence가 핵심(metadata·fencing 필요). TCP 중복 제거는 단일 connection 내만 — 재연결·HTTP POST 재시도엔 부족.
- `end-to-end argument`(Saltzer 1984): 기능은 endpoint에서만 완전 구현 가능. 해법 — client가 생성한 `request ID`를 DB까지 전달, uniqueness constraint로 중복 억제(약한 isolation에서도 동작). checksum·encryption도 end-to-end여야 완전.

### Enforcing Constraints
- `uniqueness constraint requires consensus`(단일 leader 또는 value로 sharding). async multi-leader는 불가. log 기반: 충돌 가능 write를 같은 shard로 라우팅·순차 처리(username 등).
- multishard 처리: atomic commit 없이 sharded log+stream processor로 동등한 correctness(송금 예 — 초기 event를 atomic write하면 downstream은 결국 처리, request ID로 중복 제거·deterministic).

### Timeliness vs Integrity
- `timeliness`(최신 상태 관찰, 위반은 일시적 — eventual consistency) vs `integrity`(손상 없음, 위반은 영구 — 명시적 repair 필요). 대부분 integrity가 더 중요(은행 statement 지연은 OK, 금액 오류는 치명).
- dataflow는 timeliness와 integrity를 분리(async라 timeliness 보장 없으나 integrity는 exactly-once로 보장). integrity 달성: write를 단일 message로·deterministic derivation·request ID·immutable+reprocess.
- `loosely interpreted constraint`: 재고 초과·overbooking·overdraft를 허용 후 `compensating transaction`(사과)로 보정 — 많은 business가 이미 그렇게 동작. `coordination-avoiding` 시스템(약한 timeliness, 강한 integrity, 다지역 multi-leader 가능).

### Trust, but Verify
- system model 가정(crash·network)은 확률 문제. 대규모에선 드문 일도 발생(memory·disk·network corruption, software 버그 — MySQL uniqueness·PostgreSQL serializable 버그 전례).
- `auditing`: 데이터 손상 검출. HDFS/S3는 background로 replica 비교·이동('trust but verify'). 백업 복원도 주기적 검증.
- `designing for auditability`: event sourcing은 immutable event+deterministic derivation → provenance 명확, hash로 검증, 재실행으로 검사, time-travel debugging. blockchain(Merkle tree, certificate transparency)의 암호 도구는 경량 audit에 활용 가능.

---

## Summary (핵심 정리)
- 단일 도구로 모든 use case 불가 → 여러 시스템 조합(data integration). system of record를 두고 나머지를 비동기·loosely coupled로 derive(index·view·ML model). fault 격리·evolution(reprocess) 용이.
- database를 unbundling — log 기반 component 조합. dataflow(write path/read path)로 application 설계, state 변경을 end-user device까지 push, read도 event로.
- correctness는 distributed transaction이 아니라 end-to-end request ID(idempotence)·비동기 constraint 검사로 확장 가능하게 달성. uniqueness는 consensus(sharded log) 필요하나 multishard도 atomic commit 없이 가능.
- timeliness(일시적 위반 허용)와 integrity(영구 — 더 중요) 분리 → coordination 회피·다지역 성능. loose constraint+compensating transaction. 'trust but verify' auditability(event sourcing·Merkle tree).
- 다음 연결: Ch14에서 데이터 시스템의 윤리·법(doing the right thing)을 다룸.