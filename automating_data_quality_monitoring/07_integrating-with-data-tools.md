# 07. Integrating Monitoring with Data Tools and Systems

## 챕터 개요 (3줄 요약)
- data quality monitoring은 data stack의 일부이므로, table-stakes integration(warehouse, orchestrator)과 differentiator integration(catalog, BI/MLOps)이 필수다.
- data warehouse 연동은 SQL 기반이며 dialect·scalability 차이가 크고, 여러 warehouse 간 data reconciliation에 unsupervised ML이 특히 유용하다.
- orchestrator(Airflow/dbt)에는 run/sensor/validate 함수로 DAG에 끼워넣고, catalog·BI·MLOps에는 API publishing 모델로 quality 정보를 노출한다.

---

## 1. Monitoring Your Data Stack
> enterprise data stack 여러 지점에 monitoring을 배포하며, integration은 형식이 제각각이라 robust API가 필요하다.

- stack 구성: raw source, storage(warehouse/lake), orchestration(ETL), catalog·governance, MLOps, BI/analytics.
- 한 곳이 아니라 여러 곳에 배포 — warehouse 상시 모니터링 + orchestration task로 transformation gate. check 안 하는 곳도 quality 정보 활용(data drift 시 model retrain, dashboard에 issue 표시).
- 표준 부재(warehouse마다 다름), 표준화 움직임 존재(Alation Open Data Quality Initiative, dbt Semantic Layer). 대부분 단일 도구로 표준화 안 함 → build vs buy 시 integration 공수 audit·우선순위화.

---

## 2. Data Warehouses
> warehouse(lake 포함) 연동은 table-stakes — at-rest data의 source of truth이자 downstream consumer의 공급원이다.

- 대개 단일 warehouse로 표준화 안 함 → 다중 연동 필요(Teradata legacy, Postgres/SQL Server/Oracle 등 transactional, Snowflake+Databricks 병용). 다중 warehouse 간 replication 정확성 확인에 unsupervised ML 유용.

### 2.1 Integrating with Data Warehouses
> 대부분 SQL 쿼리로 test·통계·sample 추출하나 SQL dialect·scalability 요구가 plat별로 크게 다르다.

- 예: Presto는 모든 쿼리에 time-based WHERE 필수, BigQuery는 unbiased random sampling 어려움, legacy/transactional store는 query 부하·timeout에 robust해야.
- 사용자 준비물: network connectivity(IP allowlist), **read-only 전용 service account**. 전통 warehouse는 host/port/db/user/password, Snowflake는 account ID+db, Databricks는 personal access token.
- backend: 접근 가능한 모든 queryable object(table/view/materialized view) scan·정리. metadata(갱신 시점·volume) 추출 → observability·lineage·root cause. metadata 기반 observability가 전체 ML보다 훨씬 저렴 → 대부분 table은 observability, 중요 table만 deep monitoring. object/metadata scan은 **daily** 권장(+수동 refresh REST endpoint).
- **architecture(Anomalo 예)**: 단일 VM(OCI/Docker) 또는 Kubernetes(Helm). web frontend·API(SSO·API key) → web server(HTTPS) → controller가 job queue 생성 → dynamic worker pool이 metadata/SQL/check/ML 처리. shared state는 internal Postgres(또는 RDS). worker가 notification 전송·warehouse read·대형 object를 cloud object store(S3)에 write·runtime error를 Sentry로(고객 data 제외). container registry 감시로 자동 upgrade.

### 2.2 Security
> business data source 연동은 PII 등 민감정보 보호 책임을 수반한다.

- compliance·법적 요구 준수 필요. cloud 고객 data 저장 service provider의 산업 표준은 **SOC 2** certification(AICPA).

### 2.3 Reconciling Data Across Multiple Warehouses
> 다중 storage 시 중복 data가 동일한지 확인(예: migration 전후)하는 세 방식이 있다.

- **Rule-based testing**: 두 table 동일 조건 — schema 동일(column·type·order), primary key 집합 동일(1:1 join), join 후 값 동일. 단 같은 warehouse에 있어야 하고(대규모엔 비실용), ETL이 false positive 유발.
- **Unsupervised ML**: 각 warehouse에서 sample 후 "A인지 B인지" 예측 model 학습, SHAP로 차이 root cause. row-level 전부는 못 잡아도 거대 dataset·다른 warehouse를 ETL 없이 비교.
- **Summary statistics**: row count·category별 평균 등 통계 비교 — scalable하나 전체 record 비교는 불가.
- 실무 최선: 후자 둘 결합 — 집계 row count·핵심 통계 일치 확인 + ML sample로 분포 차이 없음 확인.

---

## 3. Data Orchestrators
> in-flight(ingestion·transformation 중) 또는 warehouse 진입 전 check를 위해 Airflow/dbt/Fivetran/Prefect 등과 연동한다.

### 3.1 Integrating with Orchestrators
> monitoring을 DAG(순서 있는 task 의존 그래프)에 task로 끼워넣어 임의 지점에서 check 실행한다.

- 최소 3 함수 지원: **Run checks**(AnomaloRunCheck — 설정된 check 실행), **Job sensor**(AnomaloJobComplete — polling으로 완료 감지), **Validate checks**(AnomaloPassFail — must-pass 실패 시 exception으로 workflow 중단·알림).
- 라이브러리로 패키징 권장. 예시 흐름: `ingest_transform_data >> AnomaloRunCheck >> AnomaloJobCompleteSensor(poke) >> AnomaloPassFail(must_pass=[...]) >> publish_data`.

```python
class AnomaloRunCheckOperator(BaseOperator):
    def execute(self, context):
        api_client = AnomaloHook(anomalo_conn_id=self.anomalo_conn_id).get_client()
        table_id = api_client.get_table_information(table_name=self.table_name)["id"]
        run = api_client.run_checks(table_id=table_id)
        return run["run_checks_job_id"]
```

---

## 4. Data Catalogs
> catalog(Alation, Databricks Unity Catalog, DataHub, Atlan)는 data asset의 중앙 view를 제공하며, quality 도구와 연동은 자연스러운 진화다.

- catalog와 quality 통합은 둘 다 **data governance**의 일부(정책·접근·보호·신뢰). Alation의 Open Data Quality Initiative가 open DQ API·Data Quality 탭 제공.
- 양방향 가능: catalog 안에서 table 정보 옆에 quality 표시(context 전환 없이), monitoring 플랫폼에서 catalog로 deep link(사용량·metadata 탐색). 분석가가 수백 table 중 신뢰할 만한 것을 고를 때, 인기 table 중 검증된 것 파악에 유용.

### 4.1 Integrating with Catalogs
> table에 check 설정 여부 tagging·결과 표시·deep link가 baseline이며, API 기반 publishing 모델을 따른다.

- REST API(JSON) 예: `GET /get_checks_for_table`, `POST /run_checks`, `GET /get_run_results`.
- get_run_results 메타데이터: check ID, run ID, completion time, 생성·편집 정보, error status, evaluation message, exception, 이력, sample data(good/bad URL), result statistics, success flag, configuration, triage status.
- 가장 까다로운 부분: **table ID 매칭**(catalog별 이름 표기 차이). push/pull scheduling도 고려(스크립트→daily 자동화).

---

## 5. Data Consumers
> data를 소비하는 BI/analytics·MLOps에 quality 정보를 노출하는 것도 강력하다(아직 초기 단계).

### 5.1 Analytics and BI Tools
> Tableau 등은 dashboard data 검증 여부를 표시하기 좋은 destination이다.

- Tableau의 data quality warning API 등 활용. **metric 정의 불일치** 문제(view_count를 한 팀은 <10ms 제외, 다른 팀은 포함)는 진짜 issue가 아닌 일관성 부족. dbt metrics·**dbt Semantic Layer**로 핵심 metric 중앙 정의, monitoring 도구가 "Metrics Ready integration"으로 ingest.

### 5.2 MLOps
> SageMaker 등은 performance·latency·uptime은 보지만, model에 흘러드는 잘못된 data는 잘 못 잡는다.

- 연동 시 data drift나 NULL 등으로 retrain 필요를 경고. 모니터링 지점: **source data**(training 진입점), **feature store**(NULL%·분포·correlation 변화), **training data**(현재 snapshot vs 과거·production), **model performance**(training/inference 시간, test 성능, feature importance), **feature serving**(prediction 시점 feature를 최신 training과 비교 — training/serving skew), **predictions and business logic**(예측보다 그로 인한 business logic이 더 중요 — 갑자기 fraud 비율 급증?).

---

## Summary (핵심 정리)
- integration은 table-stakes(warehouse, orchestrator)와 differentiator(catalog, BI/MLOps)로 나뉜다.
- warehouse 연동은 read-only service account·SQL 기반이며 dialect·scalability 차이를 다뤄야 하고, 대부분 table은 metadata observability·중요 table만 deep monitoring한다.
- 다중 warehouse reconciliation은 rule·unsupervised ML·summary statistic 조합이 가장 reliable하다.
- orchestrator는 run/sensor/validate 함수로 DAG에 gate를 끼우고, catalog·BI·MLOps는 API publishing으로 quality 신호를 노출한다.
- SOC 2 등 보안·governance 책임을 함께 진다.
