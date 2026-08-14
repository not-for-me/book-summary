# 01. Getting Started with MLOps and ML Engineering

## 챕터 개요 (3줄 요약)

- Production-grade ML system을 만들기 위한 전체 ML life cycle을 experimentation과 development/staging/production 두 단계로 나눠 소개한다.
- MLOps는 software engineering + data science + automation이 결합된 skill set이며, 완전 자동화된 orchestrated pipeline이 핵심이다.
- 책 전반은 Kubeflow 기반 ML platform을 점진적으로 구축하며 OCR, movie recommender, RAG 세 프로젝트로 실습한다.

---

## 1.1 The ML life cycle

> ML project는 software project보다 훨씬 iterative하며, 첫 deployment로 끝나지 않는다.

### 1.1.1 Experimentation phase

- 대부분의 ML project는 trial and error 기반의 continuous experiment로 진행된다.
- 각 step 간 흐름은 nonlinear하고 loop가 많다 (예: model training 후 문제 발견 시 data preparation으로 회귀).
- 이런 반복을 안정적으로 다루기 위해 모든 step을 orchestrated pipeline으로 조립해 초기부터 automation을 확보한다.

**주요 step:**

- **Problem formulation**: 가장 먼저 "ML을 정말 써야 하는가"를 질문한다. 단순 heuristic으로 충분하면 ML sledgehammer를 자제. 복잡한 high-dimensional, nonlinear pattern에서는 ML이 필요. (예: OCR로 ID card 번호 추출 → fraud detection)
- **Data collection and preparation**: training/evaluation용 data 확보. 부족하면 synthetic data 고려. labeling (bounding box, annotation) 필요. 이후 training/validation/test set으로 분할.
- **Data versioning**: ML project = data + code. code뿐 아니라 data 변경도 behavior를 바꾸므로 data versioning이 reproducibility의 핵심. code는 Git으로 쉽지만 data(images, CSV, pandas frame 등)는 Git급 표준 tool이 아직 없음.
- **Model training**: 대량 data로 model parameter(weights)를 tuning해 error 최소화. 자동화 시 여러 experiment를 동시에 spin up 가능하고 parameter/artifact tracking으로 reproducibility 확보. (초심자 예상과 달리 시간 대부분을 차지하지 않음)
- **Model evaluation**: training set에 없는 dataset으로 성능 측정. precision, recall, AUC 등 metric 사용.
- **Model validation**: evaluation 통과 후 business stakeholder(개발팀과 다를 수 있음)가 기대대로 동작하는지 검증.

```text
Problem Formulation -> Data Collection/Prep -> Data Versioning
   -> Model Training -> Model Evaluation -> Model Validation
   (arrows are non-linear; loops back on failure)
```

---

### 1.1.2 Development/staging/production phase

> experimentation의 "탐색"에서 production의 "유지·개선"으로 초점이 이동하는 결정적 전환점.

- 이 단계에서 ethical constraint, security, scalability, robustness, real-time performance가 핵심 고려사항이 된다.
- experimentation phase의 semi-automated pipeline을 여기서는 완전 자동화한다.
- Trigger는 CI 또는 programmatic invocation에서 발생 → pipeline step을 순차 실행.

**주요 step:**

- **Model deployment**: 가장 단순한 방식은 model inference용 REST API. 이후 Docker containerization → AWS/GCP 배포. load testing, auto-scaling, versioning, rollback 전략 필요.
- **Model monitoring**: model은 live data에서 evaluation만큼 성능이 안 나옴. 두 부류의 metric 모니터링 필요 — system/operational metric(RPS, HTTP status code 등)과 ML-specific metric(data/model drift, churn·retention 등 business metric).
- **Model retraining**: 견고한 model도 주기적 retraining 필요. schedule 기반(예: 매월) 또는 threshold 기반(예: 대출 승인 급감)으로 trigger. training과 deployment 모두 자동화되어야 함.

```text
Trigger(CI) -> Data Versioning -> Model Training -> Model Evaluation
   -> Model Validation -> Model Deployment (REST API) -> Monitoring
   -> (metrics below threshold) -> re-trigger loop
```

---

## 1.2 Skills needed for MLOps

> MLOps는 여러 domain의 skill을 조합해야 하며, 처음부터 모두 전문가일 필요는 없다.

### 1.2.1 Required skills for ML engineers

- **Software engineering**: 핵심 기반. nontrivial system 배포 경험, debugging, performance gap 식별·수정 능력.
- **ML/data science**: backpropagation 세부까지 몰라도 되지만 TensorFlow/PyTorch/scikit-learn 등 framework에 익숙하고 새 framework를 두려움 없이 습득.
- **Data engineering**: data 관련 challenge(양질의 training data 확보)가 가장 까다로움.
- **Automation**: MLOps의 큰 부분. mistake 감소, 빠른 iteration, experiment reproducibility 확보 → 규제 compliance·auditing 시 critical → 궁극적으로 trust로 이어짐.

### 1.2.2 Prerequisites

- 복잡함은 조금씩 나눠 접근("코끼리를 한 입씩").
- 책은 먼저 Kubernetes 기초 → ML platform 구축 → 그 위에 ML system을 build하는 순서로 진행. (Kubernetes 익숙하면 1.3 skip 가능)

---

## 1.3 Building an ML platform

> 잘 설계된 ML platform이 ML service를 자신 있게 개발·배포하는 foundation이다.

### 아키텍처 개요

- data warehouse/lake, batch/streaming processor, data source 등은 platform 경계(dotted line) 밖에 위치. Spark/Flink 등 data processor는 platform 주변부라 일부만 다룸.
- 경계 안: **Kubeflow**(Kubernetes 기반 open source ML platform) 설치가 시작점. 핵심 component는 **Kubeflow Pipelines**(orchestration).
- Infrastructure는 letter, data component는 number로 태그되어 data 흐름 순서를 표현.
- Kubeflow가 부족한 부분(예: feature store 미포함)은 use case에 따라 점진적으로 확장.

### 1.3.1 Build vs. buy

- SageMaker(AWS)/Vertex AI(GCP) 같은 vendor platform이 있어도, 최소 한 번은 scratch부터 구축해보는 것이 매우 valuable.
- open source library를 통합하며 tool 간 상호작용을 깊이 이해하고 limitation 극복법을 학습.

### 1.3.2 From MLOps to LLMOps

- 전통 ML platform foundation은 LLM으로 자연스럽게 확장 (chapter 12·13).
- **LLMOps 확장 요소**: document processing, retrieval, vector database(semantic search), LLM safety guardrail, cost optimization, prompt management.
- pipeline orchestration·model deployment·monitoring skill이 LLMOps에 그대로 적용됨.

**Tool 선택 원칙**: 정답이 아닌 starting point. PoC로 며칠 실험해 적합성 판단. open source + 강한 community + production 검증된 것 위주.

### 1.3.3 Tools used in this book

- **ML pipeline automation → Kubeflow Pipelines**: 각 life cycle stage = pipeline component. component 간 data를 downstream으로 전달.
- **Feature store → Feast**: feature 공유로 재작성 방지. training·serving 양쪽 사용. 구성: feature server(REST/gRPC), feature registry(catalog), feature storage(persistence). offline(training/batch) + online(real-time) 두 mode. training-serving skew 방지가 큰 이점.
- **Model registry → MLflow**: trained model + artifact(plot, metadata, hyperparameter) tracking. staging→production promotion, model endpoint 노출.
- **Model deployment**: engineering 중심. portability(containerization), scalability(Kubernetes replica), performance(GPU/CPU/RAM), reliability. CI/CD로 자동화 → Docker build → container registry push → K8s manifest 적용.

---

## 1.4 Building ML systems

> 견고한 ML platform 위에서 image·tabular 두 유형의 real-world project로 다양한 challenge를 다룬다.

### 1.4.1 / 1.4.2 ML projects

- ML project는 linear가 아닌 highly iterative — 이전 step 재방문·가정 재고 필요.
- project 요구사항은 시간에 따라 변하며 새 tool·기법 탐색 필요.
- 핵심 MLOps concept은 거의 모든 ML project 유형·domain·stage에서 vital.

**세 프로젝트:**

- **Project 1 — OCR system**: ID card 탐지. dataset 구축 → image detector training → open source library로 초기 구현 → labeled dataset으로 fine-tune → service 배포.
- **Project 2 — Movie recommender**: tabular data. feature store, drift detection, model testing, observability 개념 설명에 유리. 이미 numerical format.
- **Project 3 — RAG documentation assistant (DakkaBot)**: LLM 기반 내부 문서 assistant. document ingestion → vector embedding → RAG pipeline → safety guardrail, cost optimization, semantic evaluation 확장.

---

## Summary (핵심 정리)

- ML life cycle은 idea→production을 이끄는 framework이며 iterative하다. 각 phase 이해가 ML 개발 복잡성 navigation의 열쇠.
- 신뢰할 수 있는 ML system은 software engineering + MLOps + data science skill의 조합을 요구하며, 한 번에 마스터하기보다 상호작용 이해가 중요.
- 잘 설계된 ML platform이 foundation이며, 이 책은 Kubeflow Pipelines(automation), MLflow(model management), Feast(feature management)를 통합해 사용.
- 세 프로젝트(OCR, movie recommender, RAG assistant)로 image·tabular·LLM data를 실습하며, 전통 MLOps는 LLMOps로 자연스럽게 확장된다.
- 다음 챕터(2장)에서 core MLOps concept을 더 깊이 다룬 뒤 platform 구축과 프로젝트 배포로 진행.