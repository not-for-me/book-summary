# 06. Productionizing ML Models

## 챕터 개요 (3줄 요약)

- ML model을 API service로 배포하는 BentoML(Bento + Yatai)과 data drift를 감지하는 Evidently를 실습한다.
- BentoML은 Dockerfile·K8s manifest·metrics 로직을 직접 짜지 않고도 model을 API로 배포·scaling하도록 추상화한다.
- Evidently는 reference vs current dataset의 통계 test로 drift를 감지하며 batch·real-time(microbatch) 두 방식으로 대시보드에 리포트한다.

---

## 6.1 BentoML as a deployment platform

> BentoML은 modeling app 요구사항을 **Bento**(통합 배포 포맷)로 packaging하고 **Yatai**로 K8s에 배포·운영·scaling한다.

- 기존 FastAPI 방식은 Dockerfile·K8s manifest·monitoring·CI/CD를 직접 구성해야 함 → data scientist에게 부담. BentoML이 end-to-end로 자동화.
- **Bento** = ML app을 담는 file archive(도시락 비유). **Yatai** = K8s 배포·scaling·monitoring 담당(UI 제공).
- 흐름: local에서 Bento build → container registry push → Yatai가 image build·배포.

### 6.1.1 Building a Bento (4단계)

1. **model 등록**: MLflow model을 local BentoML store로. `bentoml.mlflow.import_model(name, model_uri)`. `bentoml models list`로 확인.
2. **service + runner 초기화**: **service**는 serving 로직 정의. **runner**는 remote Python worker에서 실행되는 계산 단위로 독립 scaling(model prediction 병렬화). `bentoml.mlflow.get(name).to_runner()` → `bentoml.Service(name, runners=[...])`. `@svc.on_startup`으로 feature store·column list 초기화, `context.state`에 저장.
3. **service endpoint 정의**: `@svc.api(input=..., output=JSON(), route="/predict")`. user_id(entity) 입력 → Feast `get_online_features` → col_list로 dummy mapping → runner predict → OutputMapper로 label(0=<=50K, 1=>50K).
4. **bentofile.yaml**: Dockerfile보다 단순. service 참조, labels, include(파일), python requirements, docker env.

```yaml
service: "service:svc"
include: ["service.py", "production.env", "src/*.py", "requirements.txt"]
python: { requirements_txt: requirements.txt }
docker:
  env: [ENV_NAME=local, AWS_ACCESS_KEY_ID=minio, ...]
```

### 6.1.2 Building and pushing

- `bentoml build -f bentofile.yaml` → `bentoml list`(Size = app, Model Size = model). `bentoml containerize` → local `docker run` 테스트.
- Yatai port-forward + `bentoml yatai login --api-token` → `bentoml push`(model + app 모두).

### 6.1.3 Deploying a Bento

- Yatai UI: Models/Bentos tab 확인 → Deployments > Create(Bento명·tag·replica·CPU/memory 지정).
- Yatai가 K8s job(`yatai-bento-image-builder-*`)으로 image build.
- pod 2종: **API Server**(frontend, 요청 수신) + **Runner**(backend, model·inference). 독립 scaling. ingress service + Swagger 문서(/predict test) + `/metrics`(Prometheus용) 자동 생성.

```text
[Bento (local)] -> push -> [Yatai] -> K8s job builds image
                                        |
                            [API Server pod] <-> [Runner pod(s)]
                                    (/predict, /metrics, ingress)
```

---

## 6.2 Evidently for data drift monitoring

> data drift = 학습 데이터의 통계적 특성이 시간에 따라 변해 model 성능·정확도가 저하되는 현상.

### drift 유형

- **Label drift**: target(ground truth) 정의가 변함 (예: 불량품 기준 변경).
- **Prior probability shift**: class의 사전 확률 변동 (예: fraud 비율 변화).
- **Covariate shift**: input feature 분포가 변함 (예: demographics·location 변화).
- **Sudden drift**: 외부 요인(pandemic 등)에 의한 급격한 변화.

### 통계 test

- **KS Test**: 두 dataset의 CDF 비교. p-value < threshold(예: 0.05)면 drift.
- **Chi-square**: categorical 변수 분포 변화.
- **Wasserstein distance**(Earth Mover's): 분포 dissimilarity. 값이 클수록 drift 큼. Evidently 기본 threshold 0.1.
- Evidently가 데이터에 맞는 test를 자동 선택(직접 지정·custom도 가능).

### 6.2.1 Report and dashboard

- **reference dataset**(historical) + **current dataset**(예측 대상) 제공 → report 생성.
- **metric**: 핵심 구성요소(dataset/column level). **metric preset**(DataDriftPreset, DataQualityPreset, RegressionPreset) = 템플릿.
- **panel**: dashboard 시각화. DashboardPanelCounter(단일 stat), DashboardPanelPlot(line/bar/scatter/histogram).
- `evidently ui`(port 8000) → RemoteWorkspace로 workspace/project 생성 → panel 추가 → `project.save()`(필수).

```python
report = Report(metrics=[DataDriftPreset()])
report.run(reference_data=reference_data, current_data=current_data)
report.save_html("drift_report.html")
```

### 6.2.2 Batch drift detection (KFP component)

- 5장 inference pipeline의 feature retrieval와 run inference 사이에 detect_drift component 삽입. drift 감지 시 pipeline 중단.
- MLflow에서 reference dataset(4장에서 logging) 조회·MinIO에서 download, Feast의 current data와 비교. report의 metric은 dashboard와 일치해야 함. `workspace.add_report` → `project.save`.

### 6.2.3 Real-time (microbatch) drift detection

- 매 request마다 test는 부정확 → **microbatch**로 비교. window size는 앱별로 조정(예제 50).
- **MonitoringService**: 새 row를 current window에 append, window size 도달 시 report 실행, 오래된 데이터 evict(sliding window). `@svc.on_startup`에서 초기화, predict에서 `iterate(feature_df)` 호출.
- 주의: current를 training set에서 뽑으면 drift 0이어야 하나 sample=50에서 87% drift로 나옴 → 작은 sample 탓, threshold·window size 재검토 필요.

```python
def iterate(self, new_rows):
    self.current = pandas.concat([self.current, new_rows], ignore_index=True)
    if self.current.shape[0] > self.window_size:
        self.current = self.current.iloc[-self.window_size:]  # slide window
    if self.current.shape[0] < self.window_size: return       # wait more data
    self.report.run(reference_data=self.reference, current_data=self.current)
    self.workspace.add_report(project_id=self.project_id, report=self.report)
```

### drift 대응 조치

- drift 조사(유형·심각도·영향), upstream data pipeline 수정·preprocessing 개선, model retraining, feature selection 갱신.

---

## Summary (핵심 정리)

- BentoML 같은 tool은 배포의 기술 장벽을 낮춰 data scientist↔engineer 협업을 촉진하고 model→실세계 impact 경로를 가속한다.
- streamlined 배포에도 underlying 기술(Docker, Kubernetes) 이해는 커스터마이징·튜닝에 유용하다.
- data drift monitoring을 pipeline에 통합하면 성능 저하를 조기 감지해 적시 개입·비용 절감이 가능하다.
- Evidently 같은 drift detection tool은 시간에 따른 data pattern 변화를 식별해 정확도·compliance·데이터 품질을 유지한다.
- 다음 챕터(7장, Part 3)부터 실제 프로젝트에서 data analysis·preparation을 다룬다.