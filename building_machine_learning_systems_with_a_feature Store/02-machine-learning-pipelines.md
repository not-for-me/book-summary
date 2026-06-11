# 02. Machine Learning Pipelines

## 챕터 개요 (3줄 요약)
- "ML pipeline"은 모호하게 쓰이므로, 본서는 생성·수정하는 ML artifact로 명명한 구체적 pipeline(feature/training/inference)으로 엄밀히 정의한다.
- MVPS 프로세스(prediction problem→KPI→ML proxy metric→FTI 구현→UI)로 빠르게 작동하는 AI system을 만든다.
- 데이터 변환을 model-independent(MIT)·model-dependent(MDT)·on-demand(ODT) 3분류로 나눠 어느 pipeline에서 수행할지 결정한다.

---

## 1. Building ML Systems with ML Pipelines
> pipeline은 명확한 입출력 인터페이스를 가지고 schedule/연속 실행되는 프로그램이며, ML pipeline은 ML artifact를 출력한다.

- ML pipeline은 생성/수정하는 artifact로 명명: feature pipeline(features), vector-embedding pipeline(embeddings), training pipeline(model), inference pipeline(predictions), model validation/deployment pipeline 등.
- ML artifact: model, feature, training data, vector index, deployment, log. 대부분 immutable(단 feature data·vector index·deployment은 in-place 업데이트 가능).

---

## 2. Minimal Viable Prediction Service (MVPS)
> 가능한 빨리 최소 작동 AI system에 도달하는 MLOps 방법론.

- 3 기둥 식별: ① prediction problem ② 개선할 KPI ③ 가용 data source.
- prediction problem → **ML proxy metric**(KPI와 양의 상관) 매핑이 가장 어려운 단계.
- 구현(좌→우, 반복적): minimal feature pipeline(backfill+incremental) → training pipeline(custom model 필요 시) → inference pipeline → UI/dashboard.
- 예: ecommerce 추천 → KPI는 conversion/engagement → ML target은 cart 담을 확률 등.
- EDA로 데이터·품질·feature-target 의존성 파악. **kanban board**(Jira/GitHub Projects)로 data source·기술·consumer·비기능 요구(volume/velocity/freshness/SLO) 정리.

---

## 3. Writing Modular Code for ML Pipelines
> 시스템은 FTI pipeline으로, 각 pipeline 내부는 feature function으로 모듈화한다.

- 아키텍처 수준 모듈화: feature/training/inference pipeline을 **data contract**(입출력 schema + 비기능 요구) 안에서 독립 개발.
- 코드 수준: 소스를 함수/클래스로 refactor, DRY·테스트 가능하게. notebook 대신 Python module에 feature function 저장(unit test·재사용).
- **feature functions 접근**(Apache Hamilton 영감): feature마다 함수 1개로 분리 → 문서화·재사용·unit test 가능.

```python
def acquisition_cost(spend, signups):
    """Acquisition cost per user = total spend / signups."""
    return spend / signups
```

- featurized DataFrame을 feature store의 **feature group**(table)에 commit(append/update/delete). training/inference는 feature query service로 일관된 snapshot 조회.

---

## 4. A Taxonomy for Data Transformations
> 데이터 변환을 MIT·MDT·ODT로 분류해 어느 pipeline에서 수행할지 결정한다.

### Feature types
- categorical(string/enum/bool), numerical(int/float), array(list/vector embedding). 각 type별 유효 변환이 다름(categorical→encode/tokenize, numerical→normalize/scale).

### Model-Dependent Transformations (MDT)
> 모델·training data로 parameterize되는 변환(encoding, normalization 등)은 재사용 불가, training·inference pipeline 양쪽에서 수행.

- ML framework별 구현(Scikit-Learn/TF/XGBoost/PyTorch). gradient-descent 모델은 normalization 필요, decision tree는 불필요.
- feature store 이전(feature pipeline)에 하면 안 됨. training·inference에서 각각 수행 → **skew 방지** 필수.
- MDT 데이터는 EDA 어렵고($74,580 vs 0.541), feature group 저장 시 write amplification 유발(새 행이 mean/std 바꿔 전체 재계산).

### Model-Independent Transformations (MIT)
> aggregation·windowed count·RFM 등 재사용 가능한 feature를 만드는 변환. feature pipeline에서 1회 계산 후 feature store 저장.

### On-Demand Transformations (ODT)
> request 시점 데이터가 필요한 real-time feature 변환. online inference pipeline에서 UDF로 수행, 같은 UDF를 feature pipeline에서 historical 데이터에 재사용해 skew 방지.

### Taxonomy ↔ FTI 매핑
- MIT: feature pipeline에서만.
- MDT: training + inference pipeline.
- ODT: online inference + feature pipeline(batch inference는 ODT 미지원).

---

## 5. Feature Pipelines
> MIT·ODT 변환의 dataflow graph를 실행해 feature store에 feature를 생성·갱신하는 프로그램.

- 변환: 추출, validation/cleaning, feature extraction, aggregation, 차원축소(embedding), binning, feature crossing.
- 특수 유형: vector embedding pipeline, feature data validation pipeline(비동기).
- 비기능 요구: backfill/incremental 동일 처리, fault tolerance(idempotent·atomic), scalability, feature freshness, governance/security, data quality.
- 기술 선택(freshness·크기 기준): batch는 Pandas(<1GB)/Polars(수십 GB)/Spark·SQL(TB), dbt(SQL). streaming은 Feldera(저진입)/Flink(PB)/Spark Structured Streaming(Python, batch 처리라 지연↑).

---

## 6. Training Pipelines
> feature store에서 데이터를 읽어 MDT 적용·모델 학습·validation·model registry 등록·배포까지 수행.

- 4 유형: 완전 training pipeline, model deployment pipeline(별도 운영 단계, human approval·A/B·shadow), model validation pipeline(비동기, CPU로 분리해 GPU 절약), training dataset pipeline(대용량 training data를 파일로 materialize).

---

## 7. Inference Pipelines
> 새 feature data + model로 예측을 출력. batch/online/agentic 3유형.

- **batch inference**: feature store에서 precomputed feature 읽어 전체 행 예측, DB 저장. orchestrator(Airflow)로 schedule.
- **online inference**: request param + precomputed feature로 feature vector 구성 → model 호출 → prediction·feature 로깅 → 응답. deployment API, KServe/FastAPI에 model과 함께 배포.
- **agentic pipeline**: LlamaIndex/LangGraph 등으로 작성. LLM + tool. query 수신 → LLM에 tool 목록 전달 → tool 실행/결과를 loop → 최종 응답. tool은 정적 정의 또는 런타임 discover.

---

## 8. Titanic Survival as an ML System
> static Titanic 데이터를 dynamic data 버전으로 만들어 FTI pipeline로 batch/interactive ML system 구축.

- synthetic passenger 생성 함수(원 분포에서 추출 → feature drift 없음).
- feature pipeline: Pandas로 historic+new를 Hopsworks feature group에 일 1회 insert.
- training pipeline: feature view 생성 → train/test 80/20 split → XGBoost 학습 → model registry 저장.
- batch inference pipeline: 새 passenger 예측 후 로깅. counterfactual(what-if) 문제. Gradio UI 제공.
- 시작: `pip install hopsworks[python]`, Hopsworks Serverless 계정·API key(.env), 무료 tier 35GB.

---

## Summary (핵심 정리)
- AI system 구축은 feature/training/inference pipeline과 그 안의 데이터 변환에서 시작한다.
- 데이터 변환 taxonomy: MIT(재사용 feature, feature pipeline), MDT(모델별 feature, training/inference), ODT(real-time feature, online inference + feature pipeline).
- 첫 ML system으로 dynamic data 버전 Titanic survival 예측을 batch·interactive로 구축했다.
- 다음 장에서는 거주 지역의 air quality 예측 AI system을 같은 스택으로 만든다.
