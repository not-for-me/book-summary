# 02. What is MLOps?

## 챕터 개요 (3줄 요약)

- ML value는 지속적으로 개선되는 closed loop의 일부일 때 발생하며, MLOps는 이 loop를 개발·모니터링·개선하는 iterative process다.
- MLOps가 어려운 이유는 data management, 복잡한 tooling, 조직 구조, scaling, 현실 데이터의 예측 불가능성 때문이다.
- MLOps는 DevOps와 automation·CI/CD를 공유하지만 data·model 관리와 continuous training에서 근본적으로 다르며, maturity는 Level 0~2로 구분된다.

---

## 2.1 The iterative MLOps life cycle

> 실세계 ML은 problem→data→model→deploy→monitor→maintain가 순환하는 closed loop로 생각해야 한다.

### ML as a loop

- model(weights+networks)과 code(architecture+supporting code)는 별개 entity → lineage와 tracking이 sustainable workflow의 핵심.
- problem statement는 stakeholder와 잘 정의·정렬되어야 함 ("고객 churn 예측"만으로는 불충분).
- stakeholder 그룹별 핵심 질문: **Business/product**(왜 ML? target metric? business value vs 개발 노력? 오답의 impact?), **Technical**(ML이 최적해? data 저장/pipeline? 평가 metric? compute? deployment 전략? retraining trigger?), **Legal/compliance**(ethics·privacy? 민감정보 처리? governance?).

### 2.1.1 Data collection

- 이후 모든 step이 이 단계의 bias를 그대로 이어받으므로 신중한 curation 필요.
- research vs enterprise 차이: research는 검증된 open source dataset 사용, enterprise는 대개 scratch부터 구축 + deployment 환경 data로 fine-tuning(transfer learning) 필요.
- 핵심 요구사항: relevance, dataset size vs 문제 복잡도, quality, harmful bias·leakage 방지, deployment 환경을 대표하는 distribution, diversity, **lineage** (raw/중간/augmented/annotated 추적).
- lineage는 자주 간과되는 technical debt 성격 — 초기 velocity와 미래 유지보수성 trade-off. data 출처(when/where/how/why) metadata 확보, versioning을 선형 프로세스로 만들어 잘못된 data 정정 가능.
- 프로젝트 dataset: OCR → **MIDV-500**(50종 ID 문서 500 video, CVAT annotation, 주로 synthetic data라 labeling 불필요), recommender → **MovieLens 1M**(users/movies/ratings.dat).

### 2.1.2 Exploratory Data Analysis (EDA)

> 정의되지 않은 dataset 문제에 대한 first line of defense이자 개발·유지 노력 추정의 핵심.

- 목표 질문: schema는? 변할 수 있나? invalid value 방어? distribution·class balancing? robust feature? 계산 비용? cyclic 변동? 외부 요인 correlation? outlier 처리?
- multivariate analysis로 distribution·pattern 파악. 변수 많으면 PCA, t-SNE 같은 dimensionality reduction.
- **모든 assumption은 검증 전까지 risk로 취급** → data pipeline 초기에 check 삽입. 조기 pivot 가능.

### 2.1.3 Modeling and training

- model/algorithm 선택 factor: theoretical performance, input data type, 문제 class(classification/detection/regression 등).
- MLOps toolbox 핵심: model·data versioning, experiment tracking, training pipeline, hyperparameter search — 공통점은 lineage·traceability + manual step 제거.
- **modular codebase (factory pattern)**: trained model = config + data + codebase 적용의 artifact. config 파일을 Git으로 tracking → 빠른 실험(예: overfitting 시 config로 model size 축소).
- 주의: overparametrization 경계. 단순 config로 시작해 speed vs flexibility trade-off 최대화.
- 프로젝트: OCR → **YOLO**(COCO 91 class 학습, ID card는 fine-tuning 필요), recommender → 단순 baseline부터, RAG → embedding model·vector database·chat interface 추가.

```text
[data version] + [config (Git)] + [codebase] --> apply --> [model artifact]
    |________________ lineage links ________________|
```

### 2.1.4 Model evaluation

> 좋은 problem definition과 metric 선택의 중요성이 여기서 드러난다.

- metric은 domain별 적합성 다름: classification/detection → precision·recall·F1, regression → MAE·MSE. 의료는 false positive/precision 엄격, 금융은 recall 중시.
- holdout dataset은 production data의 diversity·distribution 반영, leakage 방지. 자동 평가로 human error 제거·재현성 확보. holdout도 lineage·versioning 대상.
- misclassified sample 분석 → systemic 문제·bias 식별 → data collection으로 feedback. adversarial testing, edge case simulation.
- 일부 조직은 interpretability 강제 (feature importance, decision boundary, LIME, SHAP) — 강력하지만 구현 복잡, 필요성 평가 후 도입 권장.

### 2.1.5 Deployment

- 주요 방식: **API endpoint**(microservices), **특정 hardware/edge device**(automotive, security, robotics).
- 두 방식 모두 performance·latency 중시 → optimization step(양자화 등). 최적화된 최종 버전으로 성능 테스트해야 실세계와 correlate (GPU vs embedded device 차이 주의).
- staging → production 2단계. version control·lineage로 rollback을 model 교체만으로 간단히.
- 프로젝트 tool: OCR·recommender → **BentoML**(HTTP endpoint) + Docker + **Yatai**(배포), RAG → **Chainlit**(chat interface). BentoML은 Prometheus metric(RPS, HTTP code) 기본 노출. batching, canary deployment, A/B testing 탐구.

### 2.1.6 Monitoring

- 종류: **Data monitoring**(input 통계 이상, data drift), **Performance monitoring**(accuracy 등 metric 추적), **Error monitoring**(오답 분석 → edge case 이해).
- model version control·logging이 context 제공·재현·규제 문서화 지원.
- **신뢰할 수 있는 alerting/notification 필수** — 좋은 metric도 담당자에게 도달 못하면 무의미.
- **Evidently** tool로 drift 식별·retraining trigger.

### 2.1.7 Maintenance, updates, and review

- loop를 닫는 단계: bug fix, edge case data 수집, data drift 대응 retraining.
- loop 전체에 분산된 개념 — 각 component가 고유 maintenance/update 요소 보유.
- RAG는 LLM observability(**Langfuse**), guardrail, adversarial testing, input sanitation, prompt safety, hallucination testing에 집중.

---

## 2.2 Why is robust MLOps important?

> 각 loop 블록 자체가 전문가급 난제이며, 조합될 때 복잡성이 폭증한다.

- loop의 단일 블록(예: data collection)도 대규모 구현에 specialist 필요. 각 블록이 layer를 추가.
- data at scale 관리(visibility·privacy·security)는 비싸고 litigation·재정 손실 위험. fairness·diversity·bias 제거가 복잡성 가중.
- 각 stage specialist가 tool·언어가 다름 (data scientist는 precision/recall/F1로, ML engineer는 model registry·latency·scaling으로 소통) → 공통 언어 필요.
- optimization·scaling은 초기 우선순위 아니지만 usability를 깰 수 있음. 최적화 후 성능 특성이 크게 달라짐.
- 현실의 messy data·edge case가 무증상 skew 유발 → robust monitoring·data validation 필요.
- tooling fragmentation (model registry만 10+ offering). 대개 EDA·deployment에 집중, data management·lineage는 누락.
- 작은 팀은 robust 실천을 건너뛰고 빠른 solution 선택 → 장기 technical debt 누적.

---

## 2.3 Role of MLOps in a mature organization

- **Accelerated innovation**: infra 추상화, 쉬운 versioning·비교·A/B testing으로 data scientist가 핵심에 집중.
- **Optimized costs**: silo 방지, component 공유, 중앙 전략으로 resource 관리·auto-scaling·cost monitoring.
- **Collaboration**: cross-domain 협업, visibility 증가, 명확한 소통 채널.
- **Improved efficiency**: CI/CD로 반복 작업 자동화, human error 감소, 결정론적 output.
- **Repeatability & traceability**: 진화 가시화, monitor·debug·rollback, drift·bias·regression 처리, 강한 lineage.
- **Robust scaling**: 표준 workflow·최적 resource 할당.

---

## 2.4 DevOps vs. MLOps

> 공통 원칙을 공유하나 data·model 관리에서 근본적으로 다르다.

**유사점**: robust automation, CI/CD ethos, multidisciplinary(generalist 협업).

**차이점**:

- MLOps는 model training/deploy/monitor/optimize에 집중 (DevOps는 software·infra life cycle).
- **Data**가 핵심 차별점 — MLOps는 data를 model·process의 필수 building block으로, data life cycle 관리에 큰 비중. DevOps는 이 수준의 data rigor 없음.
- model은 code 변경 없이도 계속 진화 → **Continuous Training** 추가.
- model interpretability·bias 관심, compliance monitoring.
- input data 변화로 성능 저하 → performance monitoring critical (conventional software보다 다양한 방식으로 degrade).
- 본질적으로 experimental·iterative → build가 아닌 **experiment 자체를 track**.

---

## 2.5 Levels of MLOps maturity

> Google 기준, 자동화 수준이 MLOps maturity의 지표이며 이는 개발 velocity로 직결된다.

- **Level 0: Basic** — 수동 script 기반 build/train/deploy, monitoring 없음, release 드묾.
- **Level 1: Intermediate** — 빠른 실험 + production 지속 retraining. model만이 아닌 **retraining pipeline 전체**를 deploy(continuous delivery). 팀 간 공유되는 modular pipeline component → **experimental-operational symmetry**. 필요 요소: robust data·model validation, feature store, pipeline monitoring·lineage·metadata, auto-triggered pipeline.
- **Level 2: Advanced** — pipeline 자체의 build·deploy 자동화. data analysis·model analysis 외 거의 모든 것 자동화. pipeline·component를 조직 전체가 소유, data scientist도 신규 component 기여.

---

## Summary (핵심 정리)

- ML은 business 문제 해결 수단 → 시작 전 요구사항을 깊이 이해해야 한다.
- MLOps는 ML model을 개발·모니터링·개선하는 iterative process이며, model은 성능을 점진 개선하는 loop의 artifact다.
- MLOps가 어려운 이유: data management, 복잡한 tooling, 조직 구조, scaling, 현실의 예측 불가능성.
- 확립된 실천을 건너뛰면 단기엔 빠르나 중복·technical debt로 이득이 사라진다.
- DevOps와 MLOps는 유사하나 data·model 관리, continuous training 등에서 다르다.
- 다음 챕터(3장)에서 containerization과 Kubernetes로 scalable ML platform 구축을 시작한다.