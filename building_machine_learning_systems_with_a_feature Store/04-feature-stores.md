# 04. Feature Stores

## 챕터 개요 (3줄 요약)
- feature store는 training·inference용 feature를 저장·관리·서빙하는 중앙 데이터 계층으로, FTI pipeline을 연결하고 skew를 방지한다.
- feature group(테이블)에 데이터를 저장하고, feature view로 여러 feature group의 feature를 선택해 일관된 training/inference 데이터를 만든다.
- offline store(columnar/lakehouse) + online store(row-oriented) + vector index로 구성되며, 실시간 신용카드 fraud detection이 대표 예제다.

---

## 1. A Feature Store for Fraud Prediction & History
> data source → entity/feature 식별 → feature group 구성 → feature view 선택 → training/inference 데이터의 4단계로 feature store를 설계한다.

- fraud 예제: Kafka(transaction event) + data warehouse(card/issuer/merchant) 소스, SLO ≤50ms로 fraud 판정.
- 역사: Uber Michelangelo(Palette, DSL 기반) → Hopsworks(2018, 첫 open source·API-based) → Feast → GCP/AWS/Databricks(API-based), Tecton(DSL). API-based는 DataFrame API로 read/write, orchestration 내장 안 함.
- 용어: feature platform(managed pipeline), AI lakehouse(lakehouse offline + 통합 online).

---

## 2. The Anatomy of a Feature Store
> feature store는 데이터 변환을 조직해 training·batch·online inference에 일관된 snapshot을 제공한다.

- feature pipeline이 MIT(+ODT)로 재사용 feature data 생성·저장(mutable). label도 feature data로 저장 가능.
- versioned training dataset: point-in-time snapshot + MDT 적용. lineage 저장.
- online: prediction request param으로 ODT 계산 + precomputed feature 조회 → feature vector 병합 → MDT 적용 → 예측.
- 변환 조합 규칙: MDT는 항상 DAG의 마지막(모델 호출 직전), ODT는 보통 MIT 다음. feature store가 offline/inference 간 동일 변환 보장 → skew 방지.

---

## 3. When Do You Need a Feature Store?
> real-time context/history, time-series, 협업, governance, 재사용, skew 제거, 중앙화가 필요할 때.

- **context/history**: 정보량 적은 prediction request(예: 카드번호+금액)를 history로 enrich.
- **time-series**: 여러 테이블에 흩어진 데이터로 **point-in-time correct** training data 생성(future data leakage·stale 방지) → temporal join.
- **협업(Conway's Law)**: silo를 깨는 shared platform. FTI 역할 분담(data engineer↔scientist↔ops). data scientist가 운영까지 하면 ML engineer.
- **governance**: audit, lineage(source→feature→model), PII 태그, EU AI Act 등 규제 대응.
- **discovery/reuse**: feature registry로 검색·재사용(Meta: 인기 100 feature가 100+ 모델 재사용).
- **skew 제거**: ODT/MDT를 한 번 정의해 offline·inference에 동일 적용(DRY).
- **중앙화**: offline(columnar) + online(row-oriented) + vector index 하이브리드.

### Store 구조
- **online store**(row-oriented): 저지연 CRUD, primary key 조회, TTL, ACID, secondary index.
- **offline store**(columnar/lakehouse): 저비용 대용량, 복잡 query 효율. OTF(Iceberg/Delta/Hudi) = Parquet + metadata로 ACID commit, time-travel, schema evolution.
- **vector index**: ANN search(Weaviate/Pinecone 또는 PGVector/OpenSearch/MongoDB).

---

## 4. Feature Groups
> feature group은 feature를 컬럼으로 갖는 테이블로, offline/online/vector index 저장의 복잡성을 숨긴다.

- 구성: schema + metadata(name, version, entity_id, online_enabled, event_time, tags) + offline table + (optional) online table + (optional) vector index.
- index 컬럼(entity_id, event_time, foreign_key, partition_key)은 feature 아님. label은 feature group이 아니라 모델 선택 시점에 지정.
- **untransformed data 저장**: MDT 적용 전 저장(재사용성, write amplification 방지, EDA 용이).
- feature definition: feature 생성 변환 소스코드(API-based는 MIT/ODT 코드, DSL은 선언 변환+pipeline 명세).
- **쓰기**: batch API(offline) + stream API(online_enabled, 더 fresh). CDC로 offline→online 동기화.
- **feature freshness**: ingest부터 inference에서 읽기까지 시간. real-time은 streaming pipeline 필요(예: ch15 TikTok 1초 내).
- data validation: 유효값 제약(단 missing은 나중에 impute 가능).

---

## 5. Data Models for Feature Groups
> entity→feature group, relationship→foreign key. normalization vs denormalization으로 모델링한다.

- **normalization**(중복 제거, 무결성, 읽기 join↑) vs **denormalization**(중복↑, 저장비용↑, query 단순). columnar는 denormalized 선호(columnar compression), row-oriented는 normalized 선호.
- batch는 offline만, real-time은 offline+online. 범용으로는 **snowflake schema(normalized)** 선호, 일부는 star schema만 지원.
- **dimension modeling**: facts(measured quantity) + dimensions(attribute). SCD(slowly changing dimension).
- **핵심**: feature store에서 **label = fact, feature = dimension**. label은 immutable·timestamp, feature는 mutable(SCD). 모든 시점 값 저장 필수(leakage 방지).
- feature store는 SCD Type 0(immutable, event_time 없음), Type 2(offline, event_time), Type 4(online 최신 + offline 전체)만 event_time 지정으로 단순 구현.

### 신용카드 fraud 데이터 모델
- fact: credit_card_transactions, dimension: card/account/bank/merchant_details. cc_fraud 테이블이 label(is_fraud) 제공.
- feature 예: windowed aggregation(num/sum_trans_last_10min/hour/day), ODT(haversine_distance, time_since_last_transaction), days_to_card_expiry, chargeback_rate 등.
- **star schema**: label feature group이 4개 foreign key. **snowflake schema**: 2개 foreign key → real-time에서 client가 cc_num·merchant_id만 제공하면 나머지는 subquery로 조회(star는 bank_id·account_id도 필요).

---

## 6. Data Model for Inference
> inference 시 label·label feature group은 precomputed로 없으며, request param·mapping·ODT/MDT로 채운다.

- **online inference**: request에 entity ID(foreign key) + passed feature + ODT param 포함. foreign key로 child online feature group 조회. Python/REST API.
- **batch inference**: label 없음. streaming→batch feature pipeline 대체 또는 ODT→MDT 재구현. API: 기간별 feature 읽기, 엔티티 batch 최신 feature 읽기, Spine DataFrame(star schema만).

---

## 7. Reading Feature Data with a Feature View
> feature view는 여러 feature group의 feature(+label) 선택을 추상화하며 skew 없이 training/inference 데이터를 제공한다.

- feature view 용도: point-in-time training/batch inference 데이터, foreign key로 precomputed feature 조회, MDT/ODT 적용. metadata-only(데이터 미저장).
- 같은 순서 feature + 동일 MDT 보장 → skew 방지. (Databricks FeatureLookup, Feast/Tecton FeatureService와 동의어.)
- **point-in-time correct training data**: temporal join = **ASOF LEFT JOIN**. label event_time 이하 중 가장 최근 feature 행 선택, 없으면 NULL. ASOF=future leakage 방지, LEFT=label 행 보존.

```sql
SELECT label.amount, aggs.last_week, bank.credit_rating, label.fraud
FROM cc_trans_fg AS label
ASOF LEFT JOIN cc_trans_aggs_fg AS aggs
  ON label.cc_num = aggs.cc_num AND aggs.event_ts <= label.event_ts
ASOF LEFT JOIN bank_fg AS bank
  ON aggs.bank_id = bank.bank_id AND bank.event_ts <= label.event_ts
WHERE label.event_ts > '2022-01-01 00:00';
```

- **online inference**: `feature_view.get_feature_vector(entry=[{cc_num, merchant_id}])` 한 호출로 PK lookup + left join + ODT/MDT 수행.

---

## Summary (핵심 정리)
- feature store는 AI system의 데이터 계층이다.
- feature group은 row-oriented·column-oriented·vector index 여러 store에 feature data를 저장한다.
- batch·real-time ML을 위한 data model(star/snowflake schema, SCD)을 설계했다.
- feature view는 skew 없이 training·inference용 feature data를 query한다.
- 다음 장은 구체적 feature store인 Hopsworks를 다룬다.
