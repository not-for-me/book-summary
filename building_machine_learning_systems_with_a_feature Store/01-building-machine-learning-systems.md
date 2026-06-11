# 01. Building Machine Learning Systems

## 챕터 개요 (3줄 요약)
- static dataset 학습에서 벗어나, dynamic data를 다루는 ML system(batch/real-time/agentic)을 구축하는 여정을 소개한다.
- 모든 AI system은 feature/training/inference(FTI) pipeline으로 분해되고, feature store + model registry라는 shared data layer로 연결된다.
- MLOps/LLMOps 원칙(testing, versioning, CI/CD, monitoring)이 신뢰성 있는 ML system 운영의 핵심이다.

---

## 1. The Anatomy of a Machine Learning System
> ML system은 새 데이터로 예측하는 방식(schedule vs 24/7)에 따라 batch, real-time, agentic으로 분류된다.

- **batch ML system**: 일정 주기로 batch 데이터를 읽어 예측, DB 저장(예: Spotify Discover Weekly).
- **real-time ML system**: 사용자 요청에 near real-time 예측(예: TikTok 추천).
- **agentic AI system**: 자연어 인터페이스로 high-level goal을 자율 수행(예: Lovable 코딩 어시스턴트).
- 핵심 도전: training과 inference 양쪽에서 다양한 소스의 데이터를 대규모 ML pipeline으로 처리.

---

## 2. Types of Machine Learning
> supervised, unsupervised, self-supervised, reinforcement, in-context learning이 ML system의 주 학습 유형이다.

- **supervised**: feature+label로 학습(classification, regression, LLM fine-tuning).
- **unsupervised**: label 없이 학습(예: anomaly detection).
- **self-supervised**: unlabeled에서 label 생성(masking; BERT의 masked LM·next-sentence prediction).
- **reinforcement learning(RL)**: 최적 의사결정 학습(본서 미포함).
- **in-context learning**: 충분히 큰 LLM이 prompt의 예시(context)로 새 task 학습. weight 업데이트 없음, context window 비워지면 망각. agent는 이 위에 context engineering을 더함.
- ChatGPT = self-supervised pretraining + supervised fine-tuning + RLHF + in-context learning 결합.

---

## 3. Data Sources & Mutable Data
> ML system은 다양한 소스의 mutable data를 다루며, 이는 feature engineering에 새로운 과제를 던진다.

- **데이터 소스**: tabular(row-oriented: 관계형/NoSQL, column-oriented: warehouse/lakehouse), event(Kafka), graph, unstructured(text/image/video/audio), API-scraped(SaaS).
- enterprise는 보통 operational(row) 데이터를 columnar store로 옮겨 분석 → AI의 주 소스.
- **immutable vs mutable**: ML 강의는 보통 immutable dataset(CSV/Parquet, 예: Titanic) 사용. 그러나 production은 mutable data(insert/update/delete 지원, GDPR/CCPA 대응) 사용.
- mutable data는 transformation 시점 문제를 야기: aggregation/binning은 저장 전, encoding/normalization 등 training data로 parameterize되는 변환은 저장 후(읽은 뒤) 수행.

---

## 4. A Brief History of ML Systems
> stateless online ML system의 한계를 극복하기 위해 feature store가 등장해 stateful online ML을 가능케 했다.

- 1세대: monolithic batch system(training/inference mode로 동일 코드 실행해 feature 일관성 확보) 또는 stateless online system(별도 training/serving, source 버전 관리로 일관성).
- stateless online(예: image tagging, 초기 chatbot)은 training data에 제한됨.
- 해법: history·context를 input feature로 제공. source 데이터는 client가 아닌 DB에 있음 → **feature store**(Uber Michelangelo, 2017)가 context/history를 feature로 변환·저장해 online model이 저지연으로 조회.
- **feature pipeline**: 소스 데이터를 feature로 변환해 feature store에 저장(batch 또는 stream).
- LLM: stateless chatbot의 한계 → 요청 시 context를 system prompt에 주입(**RAG** + vector DB). vector embedding pipeline으로 최신화. prompt는 LLM context window 크기로 제한. → **context engineering**.
- 진화: RAG LLM app → agent(자율적으로 데이터 소스 query, tool 사용; **MCP** 표준). ML/AI system 구분: ML은 AI의 부분집합.

---

## 5. MLOps and LLMOps
> MLOps는 신뢰성·확장성 있는 ML system을 자동화로 빠르게 개발·테스트·배포하는 실천이다.

- 기원: Google "Hidden Technical Debt in ML Systems"(2015) — 모델 학습은 작은 일부, 대부분은 데이터 관리·인프라.
- **개발 시 테스트**: unit test(feature logic), integration test(pipeline), model validation(성능·bias), evals(LLM/agent 안전·신뢰).
- **production 모니터링**: data validation, model performance monitoring, feature drift detection, A/B test, guardrails.
- MLOps 원칙: 마찰 적은 자동 테스트(CI: GitHub Actions/Jenkins), CD(dev→preprod→prod), "garbage in, garbage out"(data validation), feature·model versioning(roll back/A-B), eval로 회귀 방지, metric/dashboard/alert, 로그 기반 error analysis.
- 본서는 비전통적 접근: Terraform/Docker/Kubernetes 대신 ML pipeline의 test·version·operate·monitor에 집중.

---

## 6. A Unified Architecture: FTI Pipelines
> 모든 AI system은 feature/training/inference(FTI) pipeline으로 분해되고 feature store + model registry로 연결된다.

- modularity의 이점: 독립 개발·테스트·재사용, 팀 간 역할 분리.
- **3 pipeline**:
  - feature pipeline: data → 재사용 가능한 feature data.
  - training pipeline: feature data → trained model.
  - inference pipeline: feature data + model → predictions + logs.
- 구성: 독립 프로그램들을 **shared data layer**(feature store + model registry)로 연결.
- feature store는 real-time(row-oriented, 저지연), historical(columnar, training/batch), vector embedding(vector index) 데이터를 저장.
- 다양한 compute engine 가능: batch(SQL/Spark/Pandas/Polars/DuckDB), stream(Flink/Spark Structured Streaming/Feldera).

---

## 7. Classes of AI Systems & Frameworks
> feature store 기반 AI system은 real-time, agentic, batch, stream processing으로 나뉜다.

- **real-time(interactive)**: 요청에 예측, on-demand/precomputed feature.
- **agentic workflow**: LLM+tool로 자율적 goal 달성.
- **batch**: schedule 실행, inference store에 저장.
- **stream processing**: 사용자 입력 없이 streaming 데이터에 embedded model 예측(machine-to-machine, 예: network intrusion detection).
- edge/embedded ML(본서 미포함): network detached 기기에서 실행.
- 본서 구축 예제: air quality 예측(batch), 신용카드 fraud detection(real-time), TikTok형 video recommender(ch15), agent(LlamaIndex).
- 사용 스택: Python(Pandas/Polars, Scikit-Learn/PyTorch, KServe), Modal·GitHub Actions(serverless), Jupyter, Streamlit, **Hopsworks**(feature store/model registry/serving, 무료 tier; 저자가 개발자). Feast/MLflow/Databricks/Vertex/SageMaker로 대체 가능.

---

## Summary (핵심 정리)
- batch, real-time, LLM AI system을 feature store와 함께 소개했다.
- AI system의 주요 속성·아키텍처와 이를 구동하는 ML pipeline을 다뤘다.
- MLOps의 역사적 진화를 AI system 개발·운영의 best practice 집합으로 제시했다.
- AI system을 feature store로 연결된 FTI pipeline으로 보는 새 아키텍처를 제시했다.
