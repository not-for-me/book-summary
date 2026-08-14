# 05. Orchestrating ML Pipelines

## 챕터 개요 (3줄 요약)

- Kubeflow Pipelines(KFP)로 수동 ML step(training·feature update·inference)을 자동화·재사용 가능한 pipeline component로 전환한다.
- income classifier batch inference pipeline을 read data → retrieve features → inference → write data 4개 component로 구축한다.
- 각 component는 독립 Docker image + Kubernetes pod로 실행되며, DAG로 연결해 dependency와 data 흐름을 관리한다.

---

## 5.1 Kubeflow Pipelines: Task orchestrator

> 대부분의 inference pipeline은 data 조회 → preprocess → model 로드 → inference → 결과 저장의 공통 구조를 가진다.

- KFP는 `kfp` Python SDK로 pipeline을 정의. **component**(step)를 정의 후 조합해 pipeline 생성.
- 각 component는 자체 **Kubernetes pod**에서 실행 → 깨끗한 isolated 환경 보장. workflow의 반복성·표준화·step 간 신뢰성 있는 data 흐름 해결.

### 5.1.1 Kubeflow components

> component = ML workflow 내 재사용·조합 가능한 작업 단위. 하나의 task(preprocess/train/eval/deploy 등)를 캡슐화.

**income classifier inference pipeline의 4 component:**

- **Read data**: MinIO에서 user ID 목록 조회 → pandas DataFrame → `data_output_path`(Dataset)에 저장.
- **Feature retrieval**: Feast에서 `get_historical_features`로 feature 조회. 주의: local registry의 s3 endpoint를 `localhost:9000` → `minio-service.kubeflow.svc.cluster.local:9000`로 변경, cluster 내에서 `feast apply`(Kubernetes Job으로 실행).
- **Inference**: MLflow registry에서 model URI 조회·로드(framework별 method), `predict_proba`로 예측 → 컬럼 추가.
- **Write data**: 예측 결과를 Parquet으로 MinIO bucket에 write.

**component 구축 3단계:**

1. Python 파일로 로직 작성 (argparse로 인자 받기).
2. Dockerfile로 containerize → build/push to registry.
3. `component.yaml` 정의(name, description, inputs/outputs + type, container image, command).

**주요 개념**: output type `Dataset`으로 지정하면 KFP가 output 인식. command에서는 input은 `{inputValue}`/`{inputPath}`, output은 `{outputPath}` 사용.

```yaml
name: Read Data From MinIO
inputs:  [{name: minio_host, type: String}, ...]
outputs: [{name: data_output, type: Dataset}]
implementation:
  container:
    image: 'varunmallya/read-minio-data:latest'
    command: [python3, /app/src/read_data/read_data.py,
              --data_output_path, {outputPath: data_output}]
```

**component 설계 원칙**: Modularity(single responsibility), 명시적 inputs/outputs, Parameterization(hardcode 금지 — mlflow_host·bucket_name 등), Documentation.

**이점**: Reusability(중복 제거), Collaboration(동시 개발·전문화), Testing(개별 test로 debugging 단순화).

```text
[Read data] -> [Retrieve features] -> [Inference] -> [Write data]
   MinIO           Feast                 MLflow          MinIO
   (intermediate datasets stored in MinIO between steps)
```

### 5.1.2 Income classifier pipeline

> pipeline = 전체 ML workflow를 **DAG**(directed acyclic graph)로 표현. 방향 있는 edge, cycle 없음, task 순서·dependency 강제.

- `kfp.components.load_component_from_file`로 각 component.yaml 로드 → component object 4개(fetch_data_op 등).
- `@dsl.pipeline` 데코레이터로 pipeline 함수 정의. 함수 파라미터 = runtime 조정 가능한 pipeline parameter.
- **component 연결**: `task.outputs["data_output"]`을 다음 task의 input으로 전달 → KFP가 dependency 자동 인식 (retrieve_features는 fetch_data 완료 후 실행).
- `Compiler().compile(...)`로 pipeline 함수를 YAML로 컴파일 → KFP UI에 upload.
- 실행: **experiment** 생성(run 관리·추적·비교) → **run** 생성(runtime parameter 지정) → Start Run.

```python
@dsl.pipeline(name="income-classifier-pipeline")
def income_classifier_pipeline(minio_host, ..., feature_list, model_name, ...):
    fetch_data_task = fetch_data_op(...)
    retrieve_features_task = retrieve_features_op(
        entity_df=fetch_data_task.outputs["data_output"], ...)
    run_inference_task = run_inference_op(
        input_data=retrieve_features_task.outputs["data_output"], ...)
    write_data_task = write_data_op(
        input_data=run_inference_task.outputs["data_output"], ...)

Compiler().compile(income_classifier_pipeline, "income_classifier_pipeline.yaml")
```

- KFP는 batch inference에 적합. reproducibility·scalability·collaboration 제공, workflow를 code로 정의해 다양한 config·hyperparameter·dataset 실험.

---

## Summary (핵심 정리)

- Orchestration은 여러 복잡한 pipeline을 각 step이 필요한 data·compute를 갖도록 실행하는 것이며, Kubeflow는 KFP solution을 제공한다.
- component는 재사용·조합 가능한 작업 단위로, preprocess·train·eval·deploy 등 개별 task를 캡슐화한다.
- component는 독립적·generic하게 설계해 reusability·개발·testing·유지보수를 쉽게 한다.
- 큰 workflow를 작은 component로 분해해 modular하고 유지보수 쉬운 pipeline을 만든다.
- KFP는 SDK로 component 구축·연결·pipeline 배포/실행을 지원. 다음 챕터(6장)에서는 real-time inference·production 배포를 위한 BentoML을 다룬다.