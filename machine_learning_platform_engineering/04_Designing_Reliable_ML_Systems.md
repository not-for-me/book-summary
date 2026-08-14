# 04. Designing Reliable ML Systems

## 챕터 개요 (3줄 요약)

- ad hoc experiment를 production-ready workflow로 바꾸기 위해 MLflow(experiment tracking + model registry)와 Feast(feature store)를 실습한다.
- income classifier(<=50K vs >50K binary) 예제로 EDA → 3개 model 학습·logging → best model 선택·registry 등록의 흐름을 보여준다.
- Feast는 point-in-time correct feature set과 offline/online store를 제공해 training-serving 일관성과 feature 재사용을 보장한다.

---

## 4.1 MLflow for experiment tracking

> parameter·data·metric·artifact를 tracking해 reproducibility, model selection, performance tracking을 가능케 한다.

### tracking이 필요한 이유

- **Reproducibility**: peer와 실험 공유·재현.
- **Model selection**: 여러 architecture·metric 중 best 선택.
- **Performance tracking**: retrained model을 이전 iteration과 비교.
- 단순 Git commit/파일 공유는 수동·설명 부담 → MLflow **tracking server**가 중앙집중 관리.

### 4.1.1 Data exploration

- income data: 순수 categorical 변수(Workclass, Education, Marital-Status 등) + binary target. `pandas.describe()/info()`로 기술통계, 약 30,000 row.
- categorical vs target 분포 plot 생성 후 디렉토리에 저장.

### 4.1.2 MLflow tracking

- `mlflow ui`로 local tracking server(port 5000) 실행. `set_tracking_uri` + `set_experiment("income-classifier")`.
- **run** = ML 실험/학습 1회 실행, unique run ID로 metric·param·artifact 캡처. **experiment** = run들의 상위 project.
- `mlflow.start_run()` context 내에서 `log_artifacts`(plot/file), `log_metric`, `log_params`, `sklearn.log_model` 사용. scikit-learn/XGBoost/TensorFlow/PyTorch 지원.
- dataset은 external store(MinIO bucket)에 저장하고 `mlflow.data.from_pandas` → `log_input`으로 metadata logging (training/testing/reference context).
- 3개 model 학습: DecisionTree, RandomForest, XGBoost.
- **autolog**(`mlflow.xgboost.autolog()`): metric·artifact·param 자동 logging. 단 custom metric은 명시 필요, dataset은 array로 logging.

```python
with mlflow.start_run() as run:
    tree.fit(X_train, y_train.ravel())
    mlflow.log_metric("roc_auc_score_test", roc_auc_score_test)
    mlflow.sklearn.log_model(tree, "income-classifier")
    mlflow.log_params(tree.get_params())
```

### 4.1.3 MLflow model registry

> model life cycle 관리 repository — version 추적, governance, reproducibility.

- best model 선택 2가지 방법:
  - **MLflow UI**: search bar에 `metrics.roc_auc_score_test > 0.8` query, chart view로 비교.
  - **MLflow client**(programmatic): `get_experiment_by_name` → `search_runs`(filter+order) → model URI → `register_model`.
- 등록된 model은 inference/reproduce 시 로드, staging→production stage promotion 가능.
- **Prompts tab**(LLM prompt 관리)은 아직 제한적 → 12·13장은 Langfuse 사용. tool은 수단, 프로젝트에 맞게 선택.
- 협업하려면 local이 아닌 cloud/Kubernetes 환경에 MLflow 설치 (appendix A).

```python
run_object = mlflow_client.search_runs(
    experiment_ids=experiment.experiment_id,
    filter_string="metrics.roc_auc_score_test > 0.8",
    max_results=1, order_by=["metrics.roc_auc_score_test DESC"])[0]
model_uri = f"runs:/{run_object.info.run_id}/{experiment_name}"
mlflow.register_model(model_uri, "random-forest-classifier")
```

---

## 4.2 Feast as a feature store

> 여러 table/file에서 온 feature의 복잡한 join·point-in-time 로직을 추상화해 training/inference 일관성을 보장한다.

### 개념

- 실세계 feature는 여러 소스 → 복잡한 processing/join 필요. 예제를 demographic/relationship/occupation 3개 file로 분할(timestamp + user_id + features).
- **point-in-time correct**: 특정 시점(5/22) 조회 시 가장 최근(5/21) feature 반환. Feast가 join 로직 대신 처리.
- feature를 공통 위치(MinIO)에 두어 generation과 modeling 로직 **decouple**, 팀·조직 간 재사용·공유.
- real-time service는 low latency 필요 → offline store(MinIO)에서 online store(Redis)로 push. DynamoDB, GCP Datastore도 지원.

```text
[feature pipeline] -> [Offline store (MinIO/file)]
                              | materialize (periodic)
                              v
                      [Online store (Redis)]
[Feature registry] holds definitions; [Feast SDK] retrieves for train/inference
```

### 4.2.1 Registering features

- **Entity**: feature 수집 대상 식별자 (예: user_id). name/description/type 정의.
- **FileSource**: feature file 위치 (s3 path + endpoint override, Parquet만 지원).
- **FeatureView**: feature의 논리적 grouping. name, entities, schema(Field), **ttl**(lookback window, 365일 설정), source, tags.
- **feature_store.yaml**: project명, registry 위치, provider, offline_store(file), online_store(redis + connection string).
- `feast apply`로 entity·feature view를 registry에 등록, infrastructure 배포.

```python
demo_features = FeatureView(
    name="demographic", entities=[user],
    schema=[Field(name="Sex", dtype=String), Field(name="Race", dtype=String)],
    ttl=timedelta(days=365), source=demo_features_parquet_file_source)
```

### 4.2.2 Retrieving features

- **`get_historical_features`**(training/batch): entity df(컬럼명 = user_id, event_timestamp) + feature list. point-in-time join.
- **`feast materialize START_TS END_TS`**: offline → online store로 데이터 push.
- **`get_online_features`**(real-time): user_id의 최신 feature만 조회.

### 4.2.3 Feature server

- `feast serve`: SDK 없는 언어를 위한 REST API endpoint (POST `/get-online-features`). online store에서 feature 조회.

### 4.2.4 Feast UI

- `feast ui`(port 8888): feature view·entity·data source 시각화, tag 필터.

---

## Summary (핵심 정리)

- MLflow experiment tracker는 학습·평가 시 model performance와 hyperparameter를 tracking한다.
- MLflow model registry는 model 관리·조직화·versioning으로 협업과 배포를 지원한다.
- Feast feature store는 curated·ready-to-use feature의 관리·공유를 streamline해 개발·배포를 향상한다.
- Feast는 point-in-time join으로 inference 시점 feature 신선도를 보장하며, offline(historical)/online(low-latency) 두 store를 지원한다.
- 다음 챕터(5장)에서 Kubeflow로 이 process(MLflow↔Feast, 자동 retraining)를 pipeline으로 자동화해 scale에서 재현 가능하게 만든다.