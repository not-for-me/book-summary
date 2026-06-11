# 08. Batch Feature Pipelines

## 챕터 개요 (3줄 요약)
- batch feature pipeline은 schedule/on-demand로 소스 데이터에 MIT를 적용해 재사용 feature를 feature store에 저장하는 ETL(주로) 프로그램이다.
- batch/stream/API 소스에서 읽고, polling 또는 CDC로 incremental 처리하며, 같은 프로그램을 start/end time으로 backfill·incremental 양쪽 구동한다.
- job/workflow orchestrator로 실행하고, data contract(schema validation + Great Expectations + governance tag)로 품질을 보장한다.

---

## 1. Batch Feature Pipelines 구조
> orchestrator 트리거 → 소스 읽기(start/end time) → MIT DAG → data/schema validation → feature group 저장.

- ETL(변환 후 저장) vs ELT(저장 후 SQL 변환). batch feature pipeline은 대부분 **ETL**(Python: Pandas/Polars/PySpark; SQL보다 풍부한 feature 생성).
- batch processing은 record 단위보다 효율적, 야간 저비용·오류는 다음 run 전 수정. 단점: freshness가 interval만큼만.

---

## 2. Feature Pipeline Data Sources
> batch, stream, API 3종 소스. structured(schema) vs unstructured 구분.

- **batch source**: 관계형 DB, object store/filesystem, NoSQL(key-value Redis, document OpenSearch, JSON MongoDB, graph Neo4j, vector Weaviate). 드라이버+연결정보.
  - lakehouse는 daily partition으로 partition pruning. 대용량 backfill은 PySpark. row-oriented는 event_time 컬럼+index 필요(full table scan 회피).
- **stream source**: Kafka/Kinesis/Pub-Sub. unbounded. fresh 필요면 streaming pipeline(ch9), 아니면 batch.
- **unstructured(object store)**: text/image/video/audio 파일. batch pipeline이 파일→파일(TFRecord/augmented) + feature group 행(metadata, vector embedding)으로. LLM용 text는 chunk+embedding 저장.
- **API/SaaS**: Salesforce/HubSpot. feature pipeline엔 부적합(blocking REST) → 보통 dltHub/Airbyte로 warehouse에 먼저 적재. runtime 필요 시 ODT.

---

## 3. Synthetic Credit Card Data with LLMs
> LLM에 logical model을 주고 합성 데이터 생성 코드를 작성하게 한다.

- 신용카드 transaction 공개 데이터 없음(privacy) → LLM으로 synthetic data.
- **logical model**(ER diagram 확장): 테이블명·행수·설명, 컬럼명·타입·설명·index 유형(PK/FK/partition/event_time)·category 분포·cardinality·numerical 분포·날짜 형식·missing %.
- LLM 프롬프트로 Polars 프로그램 생성(snowflake leaf→inner→root 순), fraud transaction 추가(geographic attack, 다수 소액 결제), daily update(SCD), Kafka 연속 적재.

---

## 4. Backfilling & Incremental Updates
> backfill(historical)과 incremental(변경분)을 polling 또는 CDC로 처리.

- full load=backfill, incremental load=incremental processing. lakehouse(update/delete) 덕에 full load 감소.
- incremental 빈도: freshness SLO 충족 + 처리 용량이 도착률과 균형(OOM/과프로비저닝 회피).
- **polling**(batch 소스): last-modified 컬럼(event time), start/end time query. 단 interval 내 추가·삭제 행 누락, late-arriving 누락.
- **CDC**: system-managed timestamp/commit, change log(모든 insert/delete/update). 누락 없음 → polling보다 선호. `fg.asof(end, exclude_until=start).read()`.

### 한 프로그램으로 둘 다
- 데이터 소스를 추상화, start/end time만 다름. Hopsworks external feature group(connector+schema+event_time)로 polling.
- **time window aggregation 주의**: window 계산 위해 batch가 충분히 커야. 30일 batch + 3일 window → 28일만 계산 가능(오래된 2일은 이전 데이터 없음). start/end 조정.

---

## 5. Job Orchestrators
> 단일 프로그램을 schedule·실행·로깅·fault tolerance하는 서비스.

- 정의: 프로그램+의존성(컨테이너), 런타임(K8s/Fargate), 인자/환경변수, 리소스(CPU/GPU/memory), cron.
- **Modal**: serverless Python, 자동 컨테이너화(데코레이터로 OS/리소스/라이브러리/cron 지정), 빠른 시작, stdout 스트리밍, 초 단위 과금.
- **Hopsworks Jobs**: K8s에서 Python/PySpark, 자동 컨테이너화, base image 재사용. (Serverless엔 없음.) PySpark는 driver/executor 리소스+dynamic allocation.
- lineage: PySpark는 DAG 시각화, Polars/Pandas/DuckDB는 없음 → Hopsworks는 `parents`로 명시.

---

## 6. Workflow Orchestrators
> DAG로 여러 task 실행. task 의존성·observability·retry. 단일 프로그램이면 보통 overkill.

- ML pipeline orchestrator: Kubeflow/Metaflow/Flyte/ZenML/Vertex/Azure ML/SageMaker(대부분 training 특화, scalable feature pipeline 미지원).
- data engineering: Dagster, Prefect, Databricks Workflows, Snowflake tasks.
- **Airflow**: Python DAG, operator(Spark/Kubernetes/Hopsworks job), sensor(FileSensor/HttpSensor/ExternalTaskSensor), cron/event 스케줄.
- cloud: Azure Data Factory, AWS Step Functions, Google Cloud Composer(Airflow 기반).

---

## 7. Data Contracts & Validation
> schema validation + 품질·timeliness 보장 + governance를 feature group에 부여한다.

- schema validation(Hopsworks): 타입·길이·integrity(PK/event_time 누락) 검사.
- data contract 질문: SLO, feature 유효 범위, freshness(기대·최악), late data 허용 한도, missing % 허용.
- **governance tag**: PII 여부 등. `fg.add_tag("PII", "false")`, search로 정책 위반 검색·alert.
- **data validation(Great Expectations)**: ML은 validation을 **shift-left**(feature group 쓰기 전, 한 행이 training/inference 망칠 수 있음). (data engineering은 shift-right: WAP/medallion.)
- expectation suite 부착, `validation_ingestion_policy`(STRICT=실패 시 미저장 / ALWAYS=실패해도 저장).
- governance enforcement: NO_PII 태그 확인 → check_for_pii_data(DataProfiler/LLM) → alert.

---

## Summary (핵심 정리)
- batch feature pipeline은 batch/stream/API 소스에 MIT를 적용해 재사용 feature를 생성하고 저장 전 validation한다.
- LLM으로 신용카드 fraud data mart 합성 데이터를 생성했다.
- start/end time으로 backfill·incremental을 한 프로그램으로 처리했다.
- job/workflow orchestrator로 실행하고, data contract(validation + governance)로 SLO를 보장한다.
