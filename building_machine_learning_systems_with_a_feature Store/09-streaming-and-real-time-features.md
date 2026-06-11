# 09. Streaming and Real-Time Features

## 챕터 개요 (3줄 요약)
- streaming feature pipeline은 24/7 실행되며 event stream을 소비해 stateful 변환으로 fresh feature를 feature store에 쓴다.
- 실시간 feature는 shift-left(feature pipeline에서 precompute) 또는 shift-right(online inference에서 ODT/SQL pushdown)로 계산한다.
- windowed aggregation(tumbling/hopping/rolling)과 incremental view로 확장 가능한 fresh feature를 만들며, Flink와 Feldera를 사용한다.

---

## 1. Interactive AI Needs Real-Time Features
> 상호작용형 AI는 사용자 행동·환경 변화에 1초 내 반응해야 하며 online feature store가 RAG의 retrieval 엔진이 된다.

- 예: TikTok(swipe/like에 1초 내 반응). agent/LLM도 application ID + prompt로 실시간화.
- "지난주 주문한 신발?" → user ID로 feature store에서 관련 event 조회 → system prompt에 context로 주입.
- **feature store RAG**: application ID가 agent memory의 key. 저지연 stream processing + online feature store 필요.

---

## 2. Event-Streaming Platforms
> event stream을 중앙화하는 hub. Kafka/Kinesis/Pub-Sub. event time으로 aggregate.

- 소스: CDC/polling, activity log, sensor/IoT, application context, third-party API.
- 짧은 보존(Kafka 기본 7일). sink: event-streaming platform, lakehouse, feature store.

---

## 3. Shift Left or Shift Right?
> precompute(shift-left, feature pipeline) vs on-demand(shift-right, ODT/MDT) 선택.

- **shift-left**: 저지연 예측(P99 10ms), 성능 좋은 streaming 엔진으로 부담↓.
- **shift-right**: 지연 무관, streaming 인프라 회피, 미사용 feature 낭비 방지.
- use case: fraud/추천/predictive maintenance=shift-left, dynamic pricing/chatbot session=shift-right, PII=둘 다.

### Shift-Right Architectures
- application이 raw event를 직접 feature store에 write → online에 저장, offline 비동기 materialize.
- ODT 종류: stateless, stateful(precomputed feature), stateful(raw event DataFrame), stateful SQL(pushdown aggregation).
- pushdown SQL이 DataFrame ODT보다 저지연. 단 online store가 SQL API + **TTL** 지원해야(raw event 누적→storage 관리). 
- **TTL**: `current_time > event_time + TTL`. write 지연 시 delete도 지연 필요 → Hopsworks RonDB는 purge window 제공.

---

## 4. Shift-Left Architectures
> streaming feature pipeline로 precompute. hybrid(legacy) vs streaming-native.

- **hybrid streaming-batch(Lambda)**: stream(real-time) + batch(backfill) 별도. 공유 라이브러리로 skew 방지(Klarna). 커스텀 인프라 필요 → 비권장.
- **streaming-native(Kappa)**: 단일 stream pipeline이 real-time + historical 모두 처리.
  - 운영 모드: real-time(24/7), stream replay(historical 재생), backfilling(batch 소스, 완료 후 exit), stream reprocessing(로직 변경 재실행).
  - **event sourcing**: event stream을 object store에 장기 복사(Kafka는 단기 보존) → replay/backfill/reprocess 가능.
- streaming은 stateless+stateful 변환 모두(batch는 stateless만). 처리 보증: exactly-once / at-least-once(기본) / at-most-once.
- Hopsworks sink는 at-least-once를 exactly-once로 승격(RonDB idempotent, Hudi 중복 제거).
- **backpressure**: 소비율<생산율이면 upstream에 감속 신호(Flink→Kafka throttle).

---

## 5. Writing Streaming Feature Pipelines
> datastream(unbounded) 기반 dataflow program. operator(변환)·edge(의존)·feature group(sink).

- datastream(연속·unbounded·windowing·stateful) vs DataFrame(정적·bounded·stateless).
- 데이터 교환: forward, broadcast(설정/lookup), key-based(parallel stateful), random(stateless 부하분산).
- **stateless**(과거 무관, 병렬 쉬움) vs **stateful**(이전 event 상태 유지: rolling aggregation, session, lag, cumulative, time-since-last, windowed aggregation, stateful join).
- out-of-order data는 정상 운영의 일부로 처리.
- 엔진: Flink(분산, Java DataStream/SQL Table), Quix/Pathway(Python 단일), RisingWave(Rust 분산 SQL/Python), Spark Structured Streaming(microbatch), Feldera(Rust SQL, incremental).
- **per-event**(Flink/Feldera, subsecond) vs **microbatch**(Spark, 수십초).

### Apache Flink
- DataStream API: map, filter, keyBy(partition), reduce(incremental aggregation), window. UDF는 Java serializable. CEP 라이브러리(패턴 매칭, 예: 5분 내 10회 사용 카드 차단).

### Feldera
- SQL API: SELECT(map), WHERE(filter), GROUP BY+UDF(reduce), PARTITION BY, WINDOW TUMBLING. `RETAIN`으로 state 만료(unbounded 방지).

---

## 6. Windowed Aggregations
> window assigner·type·trigger·evaluation·sink로 구성. tumbling/hopping/rolling.

- 구성: unbounded stream, window assigner(event_time→window), window type/retention/watermark, trigger, evaluation(count/sum/max), sink.
- Flink는 session window(사용자 세션, 비활동으로 종료), global window(전체 job, 주기 emit)도 지원.
- **rolling aggregation**: 고정 window 없이 연속 이동 interval. event 도착마다 즉시 평가(최저 지연, row 보존, 메모리 집약). incremental view로 O(N)→O(1).
- **time window**: 고정 길이. **tumbling**(비중첩, 1 window/event, 종료 시 평가) vs **hopping/sliding**(hop size로 전진, 중첩 가능, event 중복).
- **watermark**: 늦은 event 허용 상한. 클수록 늦은 event 수용하나 freshness↓. late event는 offline store에만 쓰기(Flink side output, Hopsworks "late" header).
- 선택: tumbling(긴·느린·대용량·late), 가능하면 **rolling**(최신, watermark 불필요; online store write rate + incremental view 지원 시).

### Rolling Aggregations with Incremental Views
- Flink OVER aggregate는 event마다 전체 재계산(O(N), window 크기 비례) → 확장성↓.
- **incremental view**: 이전 값 재사용 + 변경분만 적용 → O(1). Feldera DBSP가 **Z-set**(count가 +/0/-, delta 표현)으로 구현.

---

## 7. Credit Card Fraud Streaming Features
> cc_num으로 group한 rolling sum/count를 incremental view로 계산하고 ASOF JOIN·lag로 enrich한다.

```sql
CREATE MATERIALIZED VIEW rolling_aggregates AS
SELECT t.cc_num, t.ts AS event_time,
  SUM(COALESCE(amount,0)) OVER window_10_minute AS sum_10min,
  COUNT(amount) OVER window_1_hour AS count_1hour, ...
FROM credit_card_transactions AS t
WINDOW window_10_minute AS (PARTITION BY cc_num ORDER BY ts
  RANGE BETWEEN INTERVAL '10' MINUTE PRECEDING AND CURRENT ROW), ...;
```

- **ASOF JOIN**(point-in-time correct): transaction을 card_details와 join해 status·account_id·bank_id enrich(real-time/backfill 모두 정확). `LEFT ASOF JOIN ... MATCH_CONDITION (t.ts >= cd.last_modified)`.
- composable transformation(nested view): invalid_card → 1-day count.
- **LAG** operator로 lagged feature(prev_ts/ip/card_present). 최종 view를 feature group sink에 write(연결: Kafka 소스 → Feldera pipeline → Hopsworks). feature group은 별도 프로그램에서 explicit schema로 사전 생성.
- (참고) Airbnb Chronon **tiled aggregation**: window를 tile로 분할, 정렬 tile(shift-left) + 미정렬 구간 on-demand(shift-right) 결합. incremental view는 전체를 shift-left.

---

## Summary (핵심 정리)
- streaming feature pipeline과 ODT로 real-time ML이 비언어 행동에 상호작용 시간 척도로 반응한다.
- shift-right(raw event 저장 후 on-demand/SQL pushdown)와 shift-left(streaming precompute)를 다뤘다.
- windowed aggregation과 window 유형, incremental view로 확장 가능한 fresh rolling aggregation을 구현했다.
- Flink와 Feldera로 streaming feature pipeline을 작성하고 fraud feature SQL을 제시했다.
