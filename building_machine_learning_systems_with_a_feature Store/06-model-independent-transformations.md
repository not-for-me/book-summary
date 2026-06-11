# 06. Model-Independent Transformations

## 챕터 개요 (3줄 요약)
- feature pipeline은 MIT(EVAC: extraction, validation, aggregation, compression)로 재사용 feature를 만들어 feature store에 저장한다.
- 데이터 양·freshness에 따라 Pandas/Polars/PySpark/SQL/stream 엔진을 선택하고, monorepo로 production 코드를 조직한다.
- DataFrame 변환을 row/column 증감 카디널리티로 분류하고, vectorized compute + Arrow zero-copy로 성능을 확보한다.

---

## 1. Source Code Organization
> AI system 전체 소스를 monorepo로 두고, pipeline·feature function·test를 디렉토리로 분리한다.

- TDD + CI/CD. notebook 너머 production 코드. `pipelines/`, `features/`(feature function), test는 별도 디렉토리(feature test=unit, pipeline test=e2e).
- **monorepo**: 공유 코드를 라이브러리화 안 해도 됨. pipeline별 `requirements.txt`로 컨테이너화.
- notebooks는 EDA용(production 아님). `uv pip install -r requirements.txt`(uv가 pip보다 빠름). **Python 의존성 버전 고정 필수**.

---

## 2. Feature Pipelines & 엔진 선택
> 데이터 양·feature freshness에 따라 stream/DataFrame/SQL 엔진을 고른다.

- 패러다임: stream processing(Python/Java/SQL), DataFrame batch(Python), data warehouse batch(SQL).
- 엔진: Pandas(<1GB), Polars(수십 GB, 멀티코어·메모리↑), PySpark(TB/PB 분산), Flink(PB stream), Feldera(SQL stream, 저진입), DuckDB(단일 SQL), dbt(SQL orchestration).
- DataFrame을 SQL보다 선호: API fetch, 광범위 cleaning, unstructured(image/video/text), Python 전용 라이브러리, custom logic.
- fresh feature 필요(real-time)면 stream, 아니면 batch(운영비↓).

---

## 3. Data Transformations for DataFrames
> 변환을 row/column 증감 카디널리티로 분류한다.

- **row size-preserving**: 컬럼 추가(feature extraction). 예: outlier 플래그, rolling_mean, conditional(when/then), temporal, rank, lag/lead.
- **row/column size-reducing**: aggregation(count/sum/mean/max/percentile, per entity), filter, compression(vector embedding/PCA). tumbling window는 행 축소, rolling window는 행 보존.
- **row/column size-increasing**: JSON explode(`unnest`), pivot, PySpark UDTF(1행→다행), cross-join.
- **join**: INNER(보존 또는 축소), LEFT OUTER(좌측 보존). feature engineering 주력.

```python
# Polars aggregation 예
df.group_by("cc_num").agg(
    pl.col("amount").filter(pl.col("category").count() > 1).max())
```

- vector embedding: embedding model로 고차원→고정 array. 대용량은 GPU. kNN search로 유사 검색.

---

## 4. DAG of Feature Functions & Lazy DataFrames
> feature function을 dataflow DAG로 조직하고, lazy evaluation으로 실행을 최적화한다.

- feature pipeline 3단계: 소스 읽기 → feature function 적용·join → feature group 쓰기. 입력으로 parametrize(backfill/incremental). selection·filter는 push down.
- **DAG**: 입력(소스), 노드(DataFrame), 엣지(feature function), 출력(feature group). 중간·leaf 노드 모두 feature group에 쓰기 가능. 변환 composition.
- **lazy DataFrame**(Polars/PySpark): action(collect/write) 시점에 실행 → 최적화. Pandas는 eager(학습엔 좋으나 성능↓).

---

## 5. Vectorized Compute, Multicore, Arrow
> native Python loop 대신 vectorized 엔진(SIMD)으로, Arrow zero-copy로 엔진 간 데이터를 옮긴다.

- native Python은 interpreted bytecode + GIL로 느림. `with_columns`, Pandas UDF 등 vectorized idiom 사용.
- 예: Python UDF apply(7.35s) vs NumPy/Polars vectorized(0.28s). Pandas 2.x는 NumPy/Arrow 백엔드.
- **Arrow**: 언어 독립 in-memory columnar. Pandas/Polars/PySpark/DuckDB가 Arrow로 zero-copy 교환. Arrow Flight(Hopsworks→Python). 단 PySpark→Pandas는 driver collect 필요(OOM 위험).

```python
arrow_table = pa.Table.from_pandas(pdf)   # zero-copy
pldf = pl.from_arrow(arrow_table)         # Polars
con.register('t', pldf.to_arrow())        # DuckDB
```

---

## 6. Data Types
> feature store가 native 타입으로 저장하고 framework 타입으로 cast하며, 정밀도 손실에 유의한다.

- feature pipeline(PySpark)과 training/inference(Pandas)가 다른 타입 체계라도 feature store가 연결.
- Hopsworks: offline=Hive 타입, online=MySQL 타입. 정밀도 손실 가능성 유의.
- 복합 타입(array/struct/map), vector embedding=float array. **tensor**(다차원, audio 1D/image 3D/video 4D)는 보통 feature store에 저장 안 하고 training/inference에서 파일→tensor 변환(또는 TFRecord로 사전처리해 GPU util↑).
- **explicit schema 권장**(CSV inference 오류·정밀도 방지). `Feature(name, type, online_type)`.

---

## 7. Credit Card Fraud Features & Composition
> LLM 프롬프트로 변환 로직 생성, 작은 window에서 큰 window aggregation을 roll-up한다.

- fraud 데이터 과제: class imbalance, nonstationary(잦은 재학습), data drift, rule-based 병행.
- LLM(Polars/Pandas/PySpark 코드 생성, Hopsworks Brewer)로 변환 작성 — 단 hallucination(예: Polars에 apply 없음) 검증·unit test 필수.
- **composition**: 1-day aggregation에서 7/30-day roll-up. count/sum=합산, max/min=전체 최대/최소, mean=weighted mean(daily count 필요), stddev=sum of squares, quantile=T-Digest, distinct count=HyperLogLog/Bloom.

---

## Summary (핵심 정리)
- feature pipeline의 MIT 작성 가이드라인을 다뤘다(monorepo, 데이터 소스, 데이터 타입).
- DataFrame 변환을 row/column 증감 기준으로 분류했다.
- Pandas/Polars/PySpark 예제와 Arrow zero-copy 데이터 이동을 살펴봤다.
- 신용카드 fraud용 MIT(binning, mapping, RFM, aggregation) 예제를 제시했다.
