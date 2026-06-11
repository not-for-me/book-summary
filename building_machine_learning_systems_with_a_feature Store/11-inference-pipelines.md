# 11. Inference Pipelines

## 챕터 개요 (3줄 요약)
- inference pipeline이 AI system 유형을 정의한다: batch(batch AI), online(real-time AI), agentic(LLM AI).
- batch는 feature view로 시간범위·entity 데이터를 읽어 PySpark로 확장하고, online은 feature store에서 context를 조회해 deployment API 뒤에 배포한다.
- embedded model(저지연·edge), KServe serving, mixed-mode UDF, failure handling, SLO로 production 추론을 구성한다.

---

## 1. Batch Inference Pipelines
> schedule로 비실시간 예측을 inference store에 저장. feature view로 inference data 조회 → MDT → predict.

- 단계: precomputed feature 조회(feature view) → MDT 적용 → `model.predict()`.
- **time range 예측**: 모델의 feature view 사용(training_dataset_version 자동 초기화 → 같은 filter 적용, skew 방지). `get_batch_data(start_time, end_time)`. (Hopsworks 외 registry는 `init_batch_scoring()` 필요.)
- **entity 예측**: 시간 partition 테이블에서 entity 최신 데이터는 full table scan 위험. `latest_features=True`, Z-order/liquid clustering index, Spine DataFrame(외부에서 entity ID 확보 시).
- **PySpark 확장**: executor마다 model 다운로드(HopsFS FUSE broadcast) → partition 읽기 → predict → inference store. XGBoost는 pickle 안 됨 → FUSE 경로 broadcast. Pandas UDF로 예측, 캐싱.
- **데이터 모델링**: lakehouse는 작은 update에 write amplification(Parquet 재작성). merge-on-read(Avro 누적)+compaction. label을 별도 child feature group(cc_fraud_fg)으로 분리하면 append만(write amplification 없음).
- **NN batch**: GPU 가속, `model.eval()`(dropout off), `torch.inference_mode()`, batch size 튜닝. MNIST 예: logits→softmax.

## 2. Batch Inference for LLMs
> prompt template로 query 구성 → LLM API → inference store. zero/single/few-shot.

- 큰 open LLM 다운로드는 GPU 부담(DeepSeek V3 671B 32bit ~2.5TB) → API 선호.
- 단계: 데이터 읽기 → prompt template(예시 포함) → LLM API 순차 호출(rate limit/cost/token quota 고려) → inference store 저장.
- tenacity로 retry/backoff, temperature(낮을수록 결정적). 응답은 별도 feature group(append 효율). fenic 같은 DataFrame LLM 라이브러리.

---

## 3. Online Inference Pipelines
> 24/7 네트워크 서비스. model 배포가 아니라 online inference pipeline 배포. feature store에서 context 조회·로깅.

- **offline-online 라이브러리 일관성**: joblib 등 버전 일치 필수. Hopsworks ODT(feature group)·MDT(feature view) 소스 자동 다운로드 + FTI base container.
- **FastAPI**: Pydantic(PredictionRequest/Response)으로 schema. 단순 모델 serving. (GPU·scaling·auth 부족.)
- **LLM deployment**: HF transformers로 model+tokenizer 다운로드 → registry. DeepSeek V3 ~700GB(.safetensors 163개). HopsFS(NVMe 캐시).
- **deployment API ≠ model signature**: 
  - **serving keys**(precomputed feature 조회), **request parameters**(ODT param 또는 passed feature). 
  - predictor script: `get_feature_vector(serving_keys, passed_features, request_parameters)`가 조회→병합→ODT→MDT→drop helper→vector 반환. `model.deploy()`로 자동 생성.
  - deployment API는 client가 의존하는 안정적 contract(SLO·p99 포함). model 버전 변경에도 client 불변. feature view만 배포도 가능.

---

## 4. Model-Serving with KServe
> Kubernetes 기반 serving. pluggable backend, A/B, multimodel, serverless(scale-to-zero), 관측성.

- transformer(전처리/후처리, CPU) + predictor(model, GPU) = **InferenceService**(CPU/GPU 분리로 GPU util↑).
- backend: TensorFlow Serving, TorchServe, ONNX Runtime, Python server(XGBoost/Scikit), **Triton**(GPU 고성능), **vLLM**(LLM). Triton/vLLM은 dynamic batching.
- Hopsworks가 logging(OpenSearch)/metrics(Prometheus)/auth/feature store/vector index 연결. vLLM은 YAML config + GPU 리소스 지정.

---

## 5. Performance & Failure Handling
> mixed-mode UDF(Python 저지연/Pandas 고처리량), native UDF, log-and-wait, fallback.

- **mixed-mode UDF**: 동적 타입으로 Python UDF(float, 저지연 online)/Pandas UDF(Series, 대용량 feature pipeline) 양쪽. 일부 로직은 mixed 불가 → 2개 구현 + unit test.
- **native UDF**(C/C++/Rust): 최저 지연이나 historical 처리 어려움 → log-and-wait(online 출력 로깅으로 training data 축적). WeChat: 작은 batch는 native, 큰 batch는 LLVM JIT vectorized.
- **failure handling**: stdout/stderr 로깅(OpenSearch). 데이터 문제(missing param/feature/RAG, ODT/MDT invalid, API timeout) → impute(mean/model), default, cached/historical, simpler model fallback.
- **SLO**: 총 지연 = feature 조회 + ODT + MDT + predict + 로깅 + 네트워크. Hopsworks 최적화: 병렬 PK lookup, LEFT JOIN/projection/aggregation pushdown(RonDB), async 로깅. RAG는 k↓, 네트워크 호출은 짧은 timeout.

---

## 6. Inference with Embedded Models
> 네트워크 호출 불가/저지연(self-driving, robot, HFT) 시 embedded/host-local model.

- real-time = 고정 시간 내 완료. embedded model(앱 패키지/registry→local disk)로 unreliable 네트워크·분산 서비스 의존 제거. 하드웨어 가속 가용성 확인.
- **embedded app**: C/C++/Rust/Go/Java(XGBoost JNI, ONNX C++ API). feature/training은 Python 유지.
- **stream-processing app**(Flink/Spark Structured Streaming): embedded XGBoost로 고처리량 예측(예: network intrusion detection). worker당 1회 model 로드(`spark.python.worker.reuse`).
- **Python UI**: Streamlit(declarative)/Gradio(function)/Taipy(JS/CSS). model을 embedded로 다운로드. `@st.cache_data`/`@st.cache_resource`로 캐싱.

---

## Summary (핵심 정리)
- batch/online/embedded/streaming inference pipeline을 다뤘다.
- batch: feature view로 시간범위·entity 데이터 조회, PySpark 확장.
- online: deployment API로 model signature를 숨기고, Python/Pandas/native UDF·failure handling·SLO로 최적화.
- LLM: API 기반 batch inference와 KServe GPU serving.
