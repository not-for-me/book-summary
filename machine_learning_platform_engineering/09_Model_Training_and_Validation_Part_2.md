# 09. Model Training and Validation: Part 2

## 챕터 개요 (3줄 요약)

- production training은 K8s PersistentVolume(PVC)로 대용량 data를 효율 관리하고, MLflow·TensorBoard로 실험 추적·lineage를 유지한다.
- PVC 도입으로 component 간 반복 download를 제거하고 코드를 대폭 단순화하며, TensorBoard로 상세 training 시각화를 한다.
- movie recommender(PyTorch matrix factorization) 프로젝트에 MLflow experiment tracking·model registry(staging→prod alias)를 적용한다.

---

## 9.1 Storing data with PersistentVolumeClaim

> PVC는 pipeline component 간 공유 persistent storage를 제공해 "한 번 download, 재사용"을 가능케 한다.

- 기존 문제: dataset 반복 download, 비효율 저장, component 간 공유 제한. MinIO는 POSIX filesystem이 아니라 file 조작이 까다로움.
- `pip install kfp[kubernetes]` → `kubernetes.CreatePVC(pvc_name, access_modes=['ReadWriteOnce'], size, storage_class_name)`(VolumeOp).
- 각 component에 `kubernetes.mount_pvc(op, pvc_name, mount_path='/data')`로 mount. Artifact와 달리 data dependency가 암시되지 않으므로 `.after(prev_op)`로 순서 명시.
- 효과: download component에서 MinIO 코드 제거하고 `/data/DATASET`에 바로 추출, split은 file을 폴더로 move, train/validate는 InputPath 인자·MinIO 코드 제거 → 대폭 단순화. (단 MinIO 방식도 여전히 유효 — data type·infra에 따라 선택.)

```python
pvc = kubernetes.CreatePVC(pvc_name='yolo-pipeline-pvc',
    access_modes=['ReadWriteOnce'], size='1Gi', storage_class_name='local-path')
download_op = download_task()
kubernetes.mount_pvc(download_op, pvc_name=pvc_name, mount_path='/data')
split_op = split_dataset(...).after(download_op)
```

---

## 9.2 Tracking training with TensorBoard

> MLflow는 high-level 실험 추적·model versioning, TensorBoard는 상세 training trace·model debugging에 강하다.

- TensorBoard 제공: 실시간 metric monitoring, visual debugging, 다중 run 비교, model architecture 시각화. Kubeflow에 native 통합.
- YOLOv8은 TensorFlow 미사용이나 TensorBoard log를 project dir에 생성 → OutputPath 대신 VolumeOp mount point를 project로 사용.
- **launch**: run의 create-pvc 완료 후 PVC name 확인 → Kubeflow sidebar > TensorBoards > New TensorBoard → PVC 선택·mount path 비움 → CREATE → CONNECT.
- YOLOv8 기본 graph: mAP graph(성능 monitoring, trend/pattern 식별, early stop, hyperparameter 최적화), model architecture(복잡도 이해).

---

## 9.3 Movie recommender project

> object detection의 infra 패턴을 recommendation system(tabular)에 적용하며 새 고려사항을 다룬다.

### 9.3.1 Read from MinIO + QA

- 사용 직전 data 가정 재검증(ingest/transform pipeline과 training pipeline 분리). schema·data type 확인.
- **data access trade-off**: local download(저장비용↑·startup 느림·training 빠름) vs MinIO 직접 read(저장비용↓·startup 빠름·network 집약, 고대역 network 필요). PyArrow로 MinIO에서 직접 read.
- QA: 필수 컬럼(user_id, item_id, rating) 존재 확인. 나중에 Evidently 등으로 교체 가능.

### 9.3.2~9.3.3 Training + Metrics

- 간단한 **matrix factorization** model(PyTorch): user/item Embedding + 2 hidden layer. SGD optimizer + StepLR scheduler + L1Loss. batched training + 매 iter test loop.
- evaluation metric은 business goal에서 도출·사전 정의(팀 간 소통·객관성). 3가지:
  - **RMSE**: 예측 rating 오차, 낮을수록 좋음(완벽=0).
  - **Precision@50**: 추천의 relevance(TP/(TP+FP)), 높을수록 좋음.
  - **Recall@50**: relevant item 포함량(TP/(FN+TP)), 높을수록 좋음.
- 주의: metric은 model 성능을 소수 숫자로 판단 → business 요구와 정렬 필수(잘못하면 좋은 model 폐기·나쁜 model 과대평가).

### 9.3.4 Experiment tracking with MLflow

> 실험이 팀·시간에 걸쳐 scale되면 "어떤 hyperparameter가 최고였나"를 systematic tracking 없이는 답하기 어렵다.

- MLflow 제공: 자동 실험 logging, 중앙 artifact 저장, 표준 model packaging, lineage tracking.
- TensorBoard(training 과정 이해) + MLflow(실험 indexing·검색·model registry) 조합이 최적. 대규모 team에 MLflow가 적합.
- **upload mode**: proxied access mode(model 저장이 tracking server 경유, 편리하나 단일 endpoint 병목) vs 직접 storage 통신(성능↑, metadata만 server로).
- 코드 통합: `mlflow.set_experiment` + `mlflow.start_run` context에서 `log_param`, `log_metric`, `log_artifact`, `pytorch.log_model(signature=...)`. warm restart(`mlflow.pytorch.load_model`), model summary·signature logging.
- validation 코드에도 통합: run ID로 model 로드(class 불필요), 완료된 experiment에도 metric logging 가능(training/eval 분리, fine-tuning 시 유용). 다중 실험 Compare, tag·description.

```python
mlflow.set_experiment(mlflow_experiment_name)
with mlflow.start_run(run_id=mlflow_run_id):
    mlflow.log_param(k, v)
    mlflow.log_metric("avg_training_loss", loss, step=train_iter)
    mlflow.pytorch.log_model(model, "model", signature=model_signature)
```

### 9.3.5 Model registry with MLflow

> model은 experimental → staging → production 단계를 거치며, registry가 version·promotion·lineage·rollback을 관리한다.

- 실험 중 model은 run의 일부로 logging → 유망 model을 unique alias(예: `dev.ml_components.recommender`)로 register(unique name·version·alias·metadata).
- production 기준 충족 시 `prod.*`로 copy, **alias**로 특정 version 지정(active). 품질 gate·빠른 rollback(alias 변경).
- 등록 방법: web UI 또는 API. `promote_model_to_staging` component: 신규 model을 현재 staging과 metric 비교(rms/precision/recall threshold) 후 `register_model` + `set_registered_model_alias`. production 승격은 보통 수동 review.

### 9.3.6~9.3.7 Pipeline + Local inference

- QA → metadata → negative sampling → aux data → train → validate → promote to staging를 KFP pipeline으로 조합. pipeline도 component가 될 수 있음(pipeline이 pipeline trigger).
- **local inference**: model registry API로 `get_model_version_by_alias(name, "prod")` → `load_model`(class·weight 다운로드 불필요). signature 덕에 쉬운 inference. user 50에 children/drama 추천(viewing history와 일치).
- 자동 data 수집·training 하에 model은 시간이 지나며 일반화 향상(성능 상한 시 architecture 변경 필요).

---

## Summary (핵심 정리)

- VolumeOp(PVC)는 training pipeline에서 대용량 dataset을 다루는 더 나은 방법으로, 매 run마다 전체 download하는 단점을 극복한다.
- model 외 확장 artifact(metric, hyperparameter set, validation 예시)도 tracking해야 production 이상 debugging이 빨라진다.
- MLflow(다중 실험 tracking·artifact 저장)와 TensorBoard(단일 run introspection)는 상호 보완적이다.
- 중앙 model·실험 시스템(MLflow)은 개발 workflow뿐 아니라 stakeholder·PM·개발자 동기화의 조직적 이점을 제공한다.
- evaluation metric은 model 성능 판단의 핵심이며 lifetime 동안 진화하므로 iterative 개선을 지원해야 하고, 수동 평가도 중요하다.
- 다음 챕터(10장)에서 production model serving·inference를 다룬다.