# 10. Model Inference and Serving

## 챕터 개요 (3줄 요약)

- 학습·검증된 model을 BentoML로 서비스화하고, MLflow model registry와 결합해 자동 배포·inference 워크플로를 만든다.
- BentoML은 framework-agnostic packaging, real-time/batch inference, K8s scaling, 내장 observability(health·Prometheus)로 배포를 단순화한다.
- BentoML+MLflow 조합, MLflow 단독, KServe 등 여러 serving 옵션을 비교하고 팀 요구에 맞게 선택한다.

---

## 10.1 Model deployment is hard

> 전통 software 배포에 ML 고유의 복잡성이 더해진다.

- ML 고유 고려사항: inference pattern(batch/real-time), model scalability, performance monitoring·logging, continuous learning(retraining), data dependency, hardware(GPU), model versioning·rollback, explainability, resource management, security·privacy.

**Self-service 배포 이점**: error 감소, early validation, resource 최적화(K8s), bottleneck 감소, collaboration. 단 production 배포 engineer를 대체하진 않음(non-prod에서 data scientist가 테스트).

---

## 10.2 BentoML: Simplifying deployment

- unified serving(framework 무관), flexible inference(real-time+batch), scalability(K8s), 내장 monitoring·logging, model management·rollback, reproducible build, adaptive microbatching, API 추상화, resource 최적화, ecosystem 통합.

---

## 10.3 A whirlwind tour of BentoML

> BentoML Service = 하나 이상의 API server + 하나 이상의 Runner 추상화.

- **API server**: HTTP server. 다중 instance로 horizontal scaling·load balancing·parallel processing. input parsing·validation.
- **Runner**: ML model을 감싼 computational unit. 실제 inference 수행. 각 Runner는 자체 Python worker에서 실행 → 병렬 가능.
- YOLOv8 service 예: 2 endpoint — `/inference`(image→JSON: name/class/confidence/box), `/render`(image→bounding box 그려진 image).

```text
Request -> [API Server(s)] --distribute--> [Runner(s) (YOLO model)]
              (parse/validate)                 (inference)
```

---

## 10.4 Executing a BentoML Service locally

### Runner + Service 구성

- `YOLOv8Runnable(bentoml.Runnable)`: 생성자에서 model 로드, `@bentoml.Runnable.method(batchable=False)`로 inference method 정의(원격 호출 가능).
- `bentoml.Runner(YOLOv8Runnable, name=...)` → `bentoml.Service("yolo_v8", runners=[...])`.
- `@svc.api(input=Image(), output=JSON())`로 endpoint. **async** 함수 + `await runner.inference.async_run([img])` — promise resolve 대기(without await 시 결과 미도착으로 error, 서버 응답성 유지).

```python
class YOLOv8Runnable(bentoml.Runnable):
    def __init__(self): self.model = YOLO("yolov8_custom.pt")
    @bentoml.Runnable.method(batchable=False)
    def inference(self, input_img):
        return json.loads(self.model(input_img)[0][0].tojson())

svc = bentoml.Service("yolo_v8", runners=[bentoml.Runner(YOLOv8Runnable)])

@svc.api(input=Image(), output=JSON())
async def invocation(input_img):
    return await yolo_v8_runner.inference.async_run([input_img])
```

- **다중 Runner**: preprocessing(grayscale) + detector, 또는 두 model 동시 테스트(`asyncio.gather`로 병렬). A/B testing에 유용.
- **/render**: `model(img, save=True, project=...)`로 결과 저장 → `save_dir`·`path`로 image 반환.
- **Observability**: BentoML이 K8s health check(`/healthz` liveness, `/readyz` readiness)와 Prometheus `/metrics`를 out-of-the-box 제공.

---

## 10.5 Building Bentos

> **Bento** = model + service code + dependency를 캡슐화한 배포 패키지.

- `bentofile.yaml`: service, include(파일), docker base_image 지정.
- `bentoml build` → 고유 **Bento tag**(`service:version`, 예 `yolo_v8:3jghhcfxvwsrnbsb`) 자동 생성. `bentoml containerize <tag>` → Docker image.
- **Bento tag** 용도: versioning, 일관된 deployment, reproducibility. Docker tag처럼 명시적 지정 권장.
- `docker run -p 3000:3000 <tag>`로 테스트.

```yaml
service: "service.py:svc"
include: ["service.py", "yolov8_custom.pt"]
docker:
  base_image: "ultralytics/ultralytics:8.0.203-cpu"
```

---

## 10.6 BentoML + MLflow inference

> MLflow(tracking·experiment, data scientist용) + BentoML(배포·monitoring, ops engineer용) 조합이 개발·테스트에 최적.

- inference service 생성자에서 MLflow client로 `get_model_version_by_alias(name, "prod")` → model URI → `bentoml.mlflow.import_model` → `mlflow.pytorch.load_model`. `@bentoml.service` + `@bentoml.api`로 `/predict`.
- 이점: 몇 command로 완전한 model server(custom API·monitoring·health·batching·GPU 처리 불필요), 자동 문서(input schema). 자동 pipeline이 create→test→deploy. staging 로컬 테스트·A/B test 워크플로. **Yatai**(K8s scaling·중앙 저장) 통합.

```python
@bentoml.service(resources={"cpu": "2"}, traffic={"timeout": 10})
class RecommenderRunnable:
    def __init__(self, registered_model_name='recommender_production'):
        current_prod = MlflowClient().get_model_version_by_alias(registered_model_name, "prod")
        bentoml.mlflow.import_model("recommender", f"runs:/{current_prod.run_id}/model")
        self.model = mlflow.pytorch.load_model(...)
    @bentoml.api
    def predict(self, user_id: int, top_k: int=10, ...) -> np.ndarray: ...
```

---

## 10.7 Using only MLflow

> MLflow 단독으로도 전체 life cycle·inference service 가능 (tool 전환 불필요).

- 3단계: train·log → registry 등록 → `mlflow models serve -m "models:/recommender_production@prod"`.
- Docker: `mlflow models build-docker -m ... --enable-mlserver`. `/health`, `/invocations` endpoint. K8s 지식 불필요 — 단순성 우선 팀에 적합. 세밀한 배포·다양한 환경·monitoring 필요 시 BentoML 병용이 낫다.

---

## 10.8 KServe: BentoML 대안

> Kubeflow 내장, ML model을 K8s microservice로 배포·serving. framework-agnostic.

- 워크플로: namespace 생성 → **InferenceService**(CRD, modelFormat·storageUri 지정) apply → URL 할당.
- K8s 지식 상당히 필요(개발자 테스트엔 제약). Kubeflow native라 vanilla MLflow/BentoML보다 out-of-the-box scalable.

**비교**: KServe(K8s-centric, 세밀한 제어, ease-of-use 낮음, experiment tracking X) / BentoML(단순·개발자 친화, K8s 불필요) / MLflow(end-to-end life cycle, experiment tracking O).

---

## Summary (핵심 정리)

- model 배포는 고유 challenge(inference pattern, scalability, monitoring, resource 관리) 이해가 핵심이다.
- BentoML은 packaging·serving 통합 framework로 배포를 단순화해 핵심 기능에 집중케 한다.
- BentoML Service는 Runner 정의 + API endpoint + async 처리로 구성하며 health·Prometheus observability를 내장한다.
- Bento 패키징으로 환경 간 배포가 쉽고 Bento tag가 versioning·관리를 제공한다.
- MLflow(lineage·tracking) + BentoML(serving) 조합, MLflow 단독, KServe 등을 ease-of-use·K8s 호환성·제어 수준에 따라 선택한다. 다음 챕터(11장)에서 monitoring·explainability를 다룬다.