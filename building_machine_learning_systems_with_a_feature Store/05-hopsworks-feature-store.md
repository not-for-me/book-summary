# 05. Hopsworks Feature Store

## 챕터 개요 (3줄 요약)
- Hopsworks는 project 기반 보안 공간에서 feature store + MLOps 플랫폼을 제공하며, ch04의 fraud 데이터 모델을 Python으로 구현한다.
- feature group은 offline(lakehouse)·online(RonDB)·vector index에 저장되며, versioning·time-travel·TTL·CDC를 지원한다.
- feature view로 feature/label을 선택·join해 training·batch·online inference 데이터를 skew 없이 생성하고, pushdown으로 query를 가속한다.

---

## 1. Hopsworks Projects & Access Control
> project는 GitHub repo처럼 RBAC를 갖춘 협업 공간으로, feature store·model registry·deployment·dataset을 담는다.

```python
import hopsworks
project = hopsworks.login()   # HOPSWORKS_API_KEY 환경변수
fs = project.get_feature_store()
```

- 디렉토리: `<proj>_featurestore.db`(offline), Training_Datasets, Models, Statistics, DataValidation, Resources 등.
- **RBAC**: data owner(admin, read/write) vs data scientist(read-only, training/model만).
- **cluster level**: project가 보안 경계(multitenant). dynamic RBAC = job은 시작한 project의 권한만. 다른 project와 read-only 공유 → **data mesh**(도메인별 데이터 소유). 개발은 owner role, staging/prod는 least privilege.

---

## 2. Feature Groups
> feature pipeline이 batch/stream API로 쓰고, feature view가 읽는 feature 테이블. offline·online·vector index 일관성을 보장한다.

- online store = **RonDB**(분산 real-time DB, MySQL NDB fork). offline = lakehouse table(Hudi/Delta/Iceberg, S3/HopsFS). external feature group(Snowflake/BigQuery/Redshift)도 가능.
- 생성: `create_feature_group()`(에러) / `get_or_create_feature_group()`(idempotent). name·version·primary_key 필수, event_time·partition_key·foreign_key·online_enabled·time_travel_format·embedding_index·expectation_suite.
- 행 식별 = primary_key + event_time(time-series). composite PK 가능. foreign key는 late binding(feature view에서 join key 지정).
- **partition_key**(Hive-style partitioning + data skipping): low-cardinality 컬럼(보통 ISO 8601 date) 추천. high-cardinality 금지(PK를 partition key로 쓰지 말 것).

### Versioning & Time-travel
- feature group 버전마다 별도 테이블. **data versioning**: append/update/delete가 **commit**(ID+timestamp)로 기록 → Git-like.
- **time-travel query**(`as_of(ingestion_time)`) vs **incremental query**(commit 시간범위 변경분). Spark 클라이언트 전용(4.x).
- **ingestion time ≠ event time**: late-arriving data 처리. training dataset 재현 시 ingestion time 기준(event time 아님)으로 정확 재현.
  - 두 ASOF 맥락: training dataset 재현=ingestion time 기준, point-in-time correct=event time 기준.
- **schema 변경**: `append_features()`는 비파괴(default_value). 타입 변경·삭제·계산 방식 변경은 breaking → 새 버전 생성(backfill 필요) 또는 같은 PK/event_time의 새 feature group(저렴) + feature view v2로 교체.

### Online Store
- 기본 offline only. `online_enabled=True`로 online 테이블 생성(interactive/real-time용). in-memory(저지연) vs on-disk(`online_disk=True`, 대용량 비용효율).
- **TTL**: event_time 포함 시 entity당 다행 저장. TTL 경과 행 삭제(~15분). PK 제약 해제(PK+event_time). leakage 방지: `label.event_time - feature.event_time > TTL`이면 null(lookback window).

### Vector index
- online_enabled feature group에 ANN(similarity) search. embedding model로 고차원→고정 array 압축(semantic 보존).
- `find_neighbors(embedding, k)`. vector index 쓰기는 online store보다 느림 → 자주 갱신되는 컬럼은 분리 권장.
- 예: cc_fraud의 explanation을 sentence-transformers(all-MiniLM-L6-v2, 384-dim)로 embed해 유사 fraud 검색.

### Offline Store (Lakehouse)
- Iceberg/Hudi/Delta. PK uniqueness는 Hudi만 강제(Delta/Iceberg는 ASOF LEFT JOIN 시 중복 행 위험). data skipping(Z-ordering, liquid clustering, Hilbert). CDC query.
- **external feature group**: 외부 store가 offline table, metadata만 Hopsworks. data mart 테이블을 feature pipeline 소스로 활용.
- **data statistics**: 기본 자동 계산(histogram, correlation, min/max/mean/std, exact_uniqueness) → EDA·drift 모니터링.
- **CDC**: `notification_topic_name`으로 Kafka 토픽에 변경 행 발행 → event-driven ML.

---

## 3. Feature Views
> feature group과 model을 잇는 인터페이스. feature/label 선택 + MDT 정의로 training/inference 데이터를 생성한다.

### Feature Selection
- label feature group 1개(최대). foreign key로 transitively join. select_features/select_all/select_except/select → Query 객체에 `join()`.
- join key 자동 매칭(동명·동타입 PK), 불일치 시 `left_on/right_on` 명시. 이름 충돌은 `prefix=`.

```python
aggs_subtree = aggs.select_features().join(bank.select_features()).join(account.select_features())
selection = labels.select_features().join(merchant.select_features()).join(aggs_subtree)
```

### MDT (transformation function)
- feature view의 선택 feature에 선언적 부착(client에서 read 후 실행). 내장(min_max_scaler) 또는 custom UDF. `TransformationStatistics`(training dataset 통계)로 parameterize. Python UDF(저지연) vs Pandas UDF(대용량).

### 생성 & serving keys
```python
feature_view = fs.create_feature_view(name='cc_fraud', query=selection,
    labels=["is_fraud"], transformation_functions=[min_max_scaler("amount")],
    inference_helper_columns=['cc_expiry_date', ...])
```
- feature view엔 PK 없고 **serving keys**(label feature group의 foreign key, 예: cc_num·merchant_id). filter로 모델군 공유.
- training_helper_columns / inference_helper_columns: feature 아닌 보조 컬럼(ODT 파라미터 등).

### Training data: DataFrames vs Files
- `train_test_split()/training_data()` → Pandas(<10GB). `create_*` → Parquet/CSV 파일(>10GB, PySpark).
- split: random(time-independent), **time-series**(start/end time, time-series는 절대 random 금지), validation set(hyperparameter tuning), **stratified**(imbalanced, fraud는 up/downsample). 
- **재현성**: training dataset ID로 정확 재현(`get_train_test_split(training_data_id=111)`).

### Batch / Online Inference
- batch: `init_batch_scoring(training_data_version)` → `get_batch_data(start_time)` → predict → `fv.log()`. 유연성 위해 **Spine Group**(serving key+timestamp, batch 전용, 가능하면 회피).
- online: `get_feature_vector(entry={cc_num, merchant_id})` (저지연). ODT는 `@hopsworks.udf`로 등록(예: days_to_card_expiry). inference helper column은 get_feature_vector에 미포함.

---

## 4. Faster Queries (Pushdown)
> projection pushdown(필요 컬럼만) + pushdown filter(partition pruning + predicate pushdown)로 data skipping.

- **projection pushdown**: feature view가 부분 컬럼만 읽음(RonDB·lakehouse 지원). Redis 등은 미지원.
- **pushdown filter**: `fv.training_data(extra_filter=fg.date=="2024-01-10")`, `fg.filter(...).read()`. 다중 feature group은 각 backend로 chain.
- multicolumn partition key는 **순서 중요**: 첫 키(date)부터 pruning, 두 번째(country)만 필터하면 pruning 안 됨.
- predicate pushdown: index 필요(RonDB B-tree, Hudi Z-order, Delta liquid clustering). Parquet file/row-group 수준 skipping(min/max zone map).
- Parquet 파일 크기 균일·적정(수십 MB~수 GB) 유지(table service가 자동 조정).

---

## Summary (핵심 정리)
- Hopsworks project와 RBAC로 feature data 접근 제어를 구현한다.
- feature group 내부: offline(lakehouse), online(RonDB), vector index.
- feature view로 training·inference 데이터를 생성한다.
- filter(projection/pushdown)로 feature store query 성능을 개선한다.
