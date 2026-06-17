# 12. Stream Processing

> Keywords: unbounded stream, event/producer/consumer/topic, messaging system(backpressure/buffer/drop), message broker(AMQP/JMS), load balancing vs fan-out, ack/redelivery/dead letter queue, log-based broker(Kafka)/offset/log compaction, CDC/event sourcing, state-stream-immutability, CEP/stream analytics/materialized view/IVM, event time vs processing time, window(tumbling/hopping/sliding/session), stream join(stream-stream/stream-table/table-table), exactly-once/microbatching/checkpointing/idempotence

## 챕터 개요 (3줄 요약)
- 데이터는 대부분 unbounded(시간에 따라 도착)라 batch의 고정 시간 분할은 느리다 — event를 발생 즉시 처리하는 것이 stream processing.
- message broker(특히 log-based)가 streaming의 filesystem 역할을 하며, database의 write도 stream(CDC/event sourcing)으로 볼 수 있다 — mutable state와 immutable event log는 동전의 양면.
- stream을 처리해 derived stream/state/view를 만들며, 시간 추론(event vs processing time, window)과 join·fault tolerance(exactly-once)가 핵심 난제.

---

## 1. Transmitting Event Streams
> `event`(불변·timestamp 포함)를 producer가 생성, 여러 consumer가 topic으로 처리. polling 대신 notification이 필요.

### Messaging System 핵심 질문
- consumer가 못 따라가면: drop / buffer(queue) / backpressure(flow control). queue가 메모리 초과 시 disk 여부.
- 노드 crash 시 손실 여부: durability(disk/replication 비용). 손실 허용 시 throughput↑.
- direct messaging(UDP multicast, ZeroMQ, webhook)은 손실 인식 필요·항상 online 가정.

### Message Broker (AMQP/JMS)
- broker = message stream 최적화 DB. producer→broker→consumer(async, buffer). DB와 차이: 전달 후 삭제(장기 저장 부적합), 짧은 queue 가정, query 제한·새 메시지 notify.
- multiple consumer: `load balancing`(메시지를 한 consumer에, 병렬 처리) vs `fan-out`(모든 consumer에). Kafka consumer group은 둘 결합.
- `acknowledgment`/`redelivery`: ack 없으면 재전달(load balancing과 결합 시 순서 뒤바뀜). 무한 재전달 loop → `dead letter queue`(DLQ).

### Log-Based Message Broker (Kafka)
- append-only log를 broker로. producer는 append, consumer는 sequential read(tail -f 유사). 처리해도 삭제 안 됨.
- `partition`(shard)별 monotonic `offset`(전순서, partition 간 순서 보장 없음). fan-out 자명, load balancing은 partition을 consumer에 할당(병렬도 = partition 수, head-of-line blocking).
- `consumer offset`: 주기적 기록(개별 ack 불필요, batching). 실패 시 마지막 offset부터(일부 중복). 같은 순서 필요한 메시지는 같은 partition(partition key).
- disk 가득 차면 segment 삭제(ring buffer, 수일~수주 buffer). object storage tiered storage(Iceberg 테이블로 batch 실행). consumer 뒤처짐은 그 consumer만 영향. 과거 메시지 replay 가능(batch처럼 derived data 재생성).

---

## 2. Databases and Streams
> DB의 모든 write는 capture 가능한 event. database와 stream의 연결은 근본적이다.

### Keeping Systems in Sync & CDC
- 여러 시스템(DB·cache·index·DW) 동기화 필요. `dual write`는 race condition(순서 역전 → 영구 불일치)·부분 실패(atomic commit 문제).
- `CDC`(change data capture): DB write 변화를 stream으로 추출 → 한 DB를 leader로, 나머지를 follower로. log-based broker가 순서 보존에 적합. Debezium(MySQL/PostgreSQL 등), Kafka Connect.
- 초기 snapshot(log 위치 연동) 필요. `log compaction`: key별 최신 값만 유지 → snapshot 없이 전체 복사. Kafka 지원.
- CDC vs event sourcing: CDC는 low-level state 변화(기존 DB에 최소 변경), event sourcing은 app-level intent event(immutable, log compaction 불가·snapshot 별도). CDC는 schema가 public API화 → `outbox pattern`/data contract.

### State, Streams, Immutability
- state = 시간에 따른 event의 결과. mutable state와 immutable event log(changelog)는 동전의 양면(state는 event stream의 적분, changelog는 state의 미분).
- immutable event 장점: 회계 ledger처럼 audit·복구(버그 시 진단 쉬움), 현재 state 이상의 정보 보존(cart 추가 후 삭제 기록), 같은 log에서 여러 read view 파생(CQRS).
- concurrency control: event를 self-contained user action으로 → 단일 append로 atomic. log가 serial order 부여(단일 thread consumer는 동시성 제어 불필요).
- immutability 한계: 높은 churn은 history 비대·compaction 부담. GDPR 삭제 vs immutability → `crypto-shredding`(키 폐기). 진짜 삭제는 어려움.

---

## 3. Processing Streams
> stream을 처리해 DB write / user push / 다른 stream 생성. batch와 유사하나 stream은 끝나지 않는다.

### Uses
- 모니터링(fraud/trading/manufacturing), `CEP`(complex event processing — event 패턴 검색, query를 장기 저장하고 event를 매칭, DB와 역할 반전), `stream analytics`(aggregation·통계, window, 확률 알고리즘 HyperLogLog/Bloom filter는 최적화일 뿐).
- `materialized view` 유지(CDC/event sourcing). `IVM`(incremental view maintenance, Materialize/RisingWave/ClickHouse — 변경분만 재계산해 freshness↑). stream search(Elasticsearch percolator).

### Reasoning About Time
- `event time`(발생 시점) vs `processing time`(처리 시점). processing time window는 backlog 처리 시 가짜 spike(deterministic 위해 event time 사용).
- `straggler event`(window 종료 후 도착): 무시 또는 correction 발행. device clock 부정확 → 3 timestamp(발생/전송/수신)로 offset 보정.
- window 종류: `tumbling`(고정·비중첩), `hopping`(고정·중첩), `sliding`(서로 N분 내), `session`(비활성까지). window는 임시 state 유지 → 용량 주의.

### Stream Joins
- `stream-stream join`(window join): 같은 session의 search-click을 window 내 매칭(click-through rate). 양쪽 index 유지.
- `stream-table join`(enrichment): activity event를 DB(user profile)로 보강. 원격 query 대신 로컬 copy(CDC로 최신 유지). table은 '시간의 시작'까지 window.
- `table-table join`(materialized view): 양쪽이 changelog. social timeline cache 유지(post/follow 변화로 갱신, (u·v)'=u'v+uv' 곱 규칙).
- 시간 의존성: stream 간 순서 미결정 시 join이 nondeterministic. `slowly changing dimension`(SCD) — 버전 ID로 deterministic(단 log compaction 불가).

### Fault Tolerance
- batch는 task 재시작·partial output 폐기로 `exactly-once`(effectively-once). stream은 무한이라 어려움.
- `microbatching`(Spark Streaming, 작은 batch) / `checkpointing`(Flink, barrier로 주기적 snapshot). 단 외부 side effect는 폐기 불가.
- `atomic commit`: 모든 output·side effect·offset 이동을 atomic하게(Kafka transaction, Dataflow). XA와 달리 framework 내부에 한정.
- `idempotence`: 여러 번 해도 결과 동일(offset 메타데이터로 중복 방지). replay 동일·deterministic·fencing 가정.
- 실패 후 state 복구: 원격 저장+replication, 로컬+주기 복제(Flink snapshot, Kafka Streams는 log-compacted topic), 또는 input replay로 재구축.

---

## Summary (핵심 정리)
- stream processing은 unbounded stream에 대한 연속 batch processing. message broker/event log가 streaming의 filesystem.
- 두 broker: AMQP/JMS(개별 메시지 ack·삭제, async RPC/task queue) vs log-based(shard별 순서·offset checkpoint·디스크 보존·replay 가능, derived state 생성에 적합).
- DB write를 stream으로(CDC 암묵적/event sourcing 명시적), log compaction으로 전체 copy 보존. derived 시스템을 changelog 소비로 최신 유지·새 view 재구축.
- 시간 추론(event vs processing time, straggler, window 4종)과 3종 join(stream-stream/stream-table/table-table)이 핵심. fault tolerance는 microbatching·checkpointing·atomic commit·idempotence로 exactly-once.
- 다음 연결: Ch13에서 streaming system의 philosophy를 다룸.