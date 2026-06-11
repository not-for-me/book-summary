# 03. Data Engineering Fundamentals

## 챕터 개요 (3줄 요약)
- ML system은 user input, system-generated, internal DB, third-party 등 다양한 data source를 다루며, 각기 다른 처리 방식이 필요하다.
- data format(row-major vs column-major, text vs binary), data model(relational, document, graph), storage engine(OLTP vs OLAP)을 용도에 맞게 선택해야 한다.
- 여러 process 간 data 전달은 database/service(REST·RPC)/real-time transport(pubsub·message queue) 세 모드가 있고, batch processing은 static feature, stream processing은 dynamic feature를 생성한다.

---

## 1. Data Sources
> **user input**: 텍스트/이미지/파일 등 사용자 직접 입력 → malformatted 가능성 높아 heavy 검증 필요, 빠른 처리 기대.
> **system-generated**: logs(memory, 호출 service 등)·model prediction·user behavior(click/scroll). logs는 malformatted 적고 주기적 처리 가능하나 volume 급증 → 신호 탐색·저장 문제(low-access storage로 비용 절감).
> **internal databases**: inventory, CRM 등 enterprise application data, ML model이 직접 사용.
> **third-party**: first/second/third-party 구분. advertiser ID(IDFA/AAID)로 수집했으나 Apple의 IDFA opt-in 전환으로 축소 → first-party data 집중.

## 2. Data Formats
> data serialization = 저장/전송 가능한 format으로 변환. 고려 요소: human-readability, access pattern, text/binary(파일 크기 영향).
> 대표: JSON(text, 어디서나, 유연하나 schema 변경 어렵고 용량 큼), CSV(text, row-major), Parquet(binary, column-major), Avro/Protobuf/Pickle.

## 3. Row-Major Versus Column-Major
> CSV는 row-major(한 row의 연속 원소가 메모리 인접) → row 접근(예: 오늘 수집한 모든 example)·write가 빠름.
> Parquet는 column-major → column 기반 read(예: 1000 feature 중 4개만)가 빠름. pandas DataFrame은 column-major(NumPy ndarray는 기본 row-major) → row 단위 접근 시 느림.

```
row-major (CSV):     fast row reads, fast writes
column-major (Parquet): fast column reads, compact
```

## 4. Text Versus Binary Format
> text(CSV/JSON)는 human-readable하나 큼, binary(Parquet)는 compact. 1000000을 text 7 bytes vs int32 binary 4 bytes.
> 예: interviews.csv 14MB → Parquet 6MB. AWS: Parquet는 text 대비 unload 2x 빠르고 storage 6x 절약.

## 5. Data Models — Relational
> 1970년 Codd의 relational model: data를 relation(tuple 집합=table)으로 조직, row/column 순서 무관.
> normalization(1NF, 2NF...): 중복 제거·integrity 향상(Book에서 Publisher 분리). 단점은 data가 여러 relation에 분산 → join 비용.
> SQL은 declarative language(원하는 output 명시, 실행 방법은 system이 결정). query optimizer가 가장 빠른 실행 계획 탐색(ML로 개선 가능). declarative ML(Ludwig, H2O AutoML)도 등장하나 feature engineering·data shift 같은 진짜 난제는 미해결.

## 6. Data Models — NoSQL (Document & Graph)
> **document model**: 자체 완결적 document(JSON/XML/BSON), schemaless(구조 가정 책임을 read 측으로 이동), locality 우수(한 book 정보를 한 document에). 단, document 간 join은 비효율.
> **graph model**: node + edge(관계 명시), 관계 기반 조회가 빠름(예: "USA에서 태어난 모두" traversal). 관계 중심 query에서 relational/document보다 유리.

## 7. Structured Versus Unstructured Data
> structured: predefined schema → 분석 쉬우나 schema 변경 시 전체 갱신 부담. data warehouse에 저장.
> unstructured: schema 미준수(주로 text, 빠른 arrival, 모든 source 처리), 구조 가정 책임은 downstream으로. data lake에 raw 저장.

## 8. Storage Engines — Transactional vs Analytical
> **OLTP**(transactional): insert/update/delete 단위, low latency·high availability 요구, ACID(atomicity, consistency, isolation, durability) (또는 더 느슨한 BASE), 주로 row-major.
> **OLAP**(analytical): column 가로지른 aggregation(예: "9월 SF 평균 가격") 효율적.
> 두 용어는 outdated: ① 경계 소멸(CockroachDB, DuckDB), ② storage-compute decoupling(BigQuery/Snowflake), ③ "online"은 online/nearline/offline 속도를 의미.

## 9. ETL: Extract, Transform, Load
> **Extract**: source에서 추출·검증, 불량 data reject. **Transform**: join/clean/표준화/dedup/feature 파생(처리의 핵심). **Load**: target(DB/warehouse)에 적재 방식·빈도 결정.
> ELT(load 먼저, 처리 나중)는 data lake에 raw 저장 → 빠른 arrival이나 대량 raw 검색 비효율. Databricks/Snowflake의 lakehouse가 lake 유연성 + warehouse 관리성 결합.

## 10. Modes of Dataflow
> **database 경유**: 가장 단순하나 양 process가 같은 DB 접근 필요·read/write 느림.
> **service 경유(request-driven)**: process를 service로 노출, REST(public API에 우세)/RPC(같은 조직 내 service 간) 요청. microservice architecture와 결합(예: Lyft의 driver/ride/price service).
> **real-time transport 경유(event-driven)**: broker(event bus, in-memory)로 service 간 결합도↓. pubsub(topic 구독, retention policy; Kafka/Kinesis)과 message queue(의도된 consumer; RabbitMQ). request-driven은 logic-heavy, event-driven은 data-heavy system에 적합.

## 11. Batch Processing Versus Stream Processing
> storage에 들어간 historical data는 batch job(MapReduce/Spark)으로 처리 → static(batch) feature(예: driver rating).
> real-time transport의 streaming data는 stream processing(Flink/KSQL/Spark Streaming)으로 처리 → dynamic(streaming) feature(예: 현재 가용 driver 수). low latency 가능, stateful computation으로 중복 제거.
> 대부분 batch+streaming feature 모두 필요. "batch는 stream processing의 special case"라는 주장도 있음.

---

## Summary (핵심 정리)
- data를 미래 사용 패턴에 맞춰 저장해야 하며, row-major(CSV, write/row read) vs column-major(Parquet, column read), text vs binary(compact)의 trade-off를 이해해야 한다.
- relational(SQL, declarative)·document(locality, schemaless)·graph(관계 중심) 세 model은 각기 다른 task에 적합하고, structured/unstructured의 경계는 "구조 가정 책임이 write 측인가 read 측인가"로 갈린다.
- storage engine은 OLTP/OLAP로 나뉘나 경계가 흐려지고 storage-compute가 decouple되며, process 간 dataflow는 database/service/real-time transport 세 모드로 이루어지고 batch는 static, stream은 dynamic feature를 만든다.
