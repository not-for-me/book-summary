# 10. Training Pipelines

## 챕터 개요 (3줄 요약)
- model-centric AI(아키텍처·hyperparameter)와 data-centric AI(feature·데이터 선택)의 균형이 좋은 training pipeline의 핵심이다.
- feature store에서 label·feature를 연결해 feature view로 reproducible training data를 만들고, XGBoost부터 LLM fine-tuning까지 학습한다.
- 분산 학습(Ray, ring all-reduce, LoRA/QLoRA)과 model evaluation·bias test·model card로 신뢰성을 확보한다.

---

## 1. Unstructured Data & Labels in Feature Groups
> supervised는 label 필요, self-supervised/unsupervised는 불필요(주로 unstructured).

- **self-supervised**: LLM(next token, autoregressive), BERT(masked LM, 비autoregressive). 
- **unsupervised**: kNN/ANN, GAN(generator vs discriminator, anomaly detection), stable diffusion.
- feature group은 unstructured 파일 metadata+경로를 index(파일 이동/삭제 시 linkage 깨짐 주의).
- **supervised label**: 이미 있으면 import, 없으면 생성(예: cc_fraud LEFT JOIN transactions → fillna(0)). weak supervision(noisy label 대량, Cleanlab).
- LLM 학습: **instruction dataset**(JSONL: instruction/input/output)으로 SFT, **preference dataset**(선호 응답)으로 RLHF reward model. feature group이 JSONL보다 index·search·time-travel·lineage 유리.

---

## 2. Root and Label Feature Groups
> feature view는 root feature group에서 graph traversal로 도달 가능한 feature/label만 포함.

- label은 feature group이 아니라 feature view에서 지정. label feature group은 root 또는 child.
- 도달 불가 feature group(예: Weather)은 foreign key 추가(예: ip→city) 후 join.
- **medallion**(bronze raw → silver 정제 3NF → gold data mart). feature는 모든 layer에서 올 수 있음.

---

## 3. Feature Selection
> 예측력 있는 feature 선택, redundant/irrelevant/prohibited/infeasible 회피.

- redundant(correlation matrix), irrelevant, prohibited(PII tag), infeasible(leaky, SLO 초과).
- 방법: recursive add/elimination, filter(mutual information), wrapper(RFE), embedded(regularization).
- **LLM 기반 feature selection**: feature 설명·통계를 prompt에 넣어 제안(bias 주의).

---

## 4. Training Data
> in-memory DataFrame(Pandas, <메모리), 파일(CSV/Parquet/JSONL/TFRecord), unstructured 파일.

- Pandas는 파일 크기의 2~3배 RAM 필요. PySpark는 toPandas() OOM 위험, Polars는 Scikit-Learn 미지원.
- 파일 포맷: CSV(row, 비splittable), Parquet(columnar, splittable, 압축), JSONL(LLM), TFRecord(splittable binary), HDF5/NPY(비splittable).
- **training dataset pipeline 분리** 이유: CPU-bound(GPU 유휴), training data > 메모리(data loader streaming).
- **splitting**: random vs **time-series**(fraud, train_end~test_start 간격으로 rolling overlap 회피), train/val/test(70-80/10-20/10-20), **stratified**(imbalanced), k-fold CV.
- **reproducible**: `training_dataset_version`(random seed + commit ID 저장). deep learning은 random seed 설정(weight init/augmentation/CUDA/dropout/shuffle).

---

## 5. Model Training
> feature 선택→architecture→hyperparameter tuning→fit→evaluate→registry의 반복 과정.

### Architecture
- 구조 데이터: <10M rows는 XGBoost(decision tree), 10-100M은 경우에 따라, >100M은 NN. tree는 해석가능·전처리 최소(NeurIPS 2022: 작은 tabular는 tree 우세).
- unstructured: CNN(image/video), Transformer(NLP/time-series), feed-forward(tabular), LSTM(sequential).
- 예: MNIST feed-forward NN(nn.Module, forward pass, cross-entropy loss + Adam optimizer). model registry에 weight+hyperparameter+transformer 저장(skew 방지).
- **checkpoint**: 주기적 저장으로 실패 복구(Llama 3는 3시간마다 실패, checkpoint로 90% 효율). distributed storage(S3/HopsFS).

### Hyperparameter Tuning (Ray Tune)
- search space + search algorithm(random/grid/Bayesian/Optuna) + scheduler(ASHA: 작은 budget 병렬→prune→promote).
- AutoML: auto-sklearn(단일), Ray Tune(클러스터). experiment tracking(MLflow/wandb).

### Distributed Training (Ray)
- **data-parallel**(모델 복제, ring all-reduce gradient sync), **tensor parallelism**(큰 tensor 분할, Megatron-LM), **model-parallel**(모델이 GPU에 안 맞을 때, DeepSpeed ZeRO-3).
- Ray Train + Ray Data(병렬 데이터 로딩, zero-copy, global shuffle) + Ray Tune. actor 기반.

### PEFT of LLMs
- LLM 3단계: pretraining(self-supervised, perplexity), SFT(instruction dataset), preference alignment(RLHF: preference data→reward model→PPO).
- **PEFT**: base weight 동결, 작은 adapter만 학습. **LoRA**(attention의 query/value에 low-rank 행렬 추가, low intrinsic dimension), **QLoRA**(4-bit base, 메모리↓). fine-tuning은 새 지식 추가엔 약함(RAG/prompt 사용).

### Credit Card Fraud (XGBoost)
- <1M sample은 tree 우세. class imbalance → up/downsample. 1-2ms 추론. max_depth/n_estimators+early stopping, learning_rate↓+GPU(gpu_hist), lambda/alpha/min_child_weight로 overfit 제어. Ray Tune으로 f1 최적화.

### 분산 학습 bottleneck
- ring all-reduce(워커 ring, upload/download 대역 사용). GPU util 점검(nvidia-smi) → NVLink → PCIe → local NVMe(iostat) → network(bmon/iftop) → object store 읽기.

---

## 6. Model Evaluation & Validation
> test data로 성능, evaluation data slice로 bias 검증. training pipeline 또는 별도 validation pipeline.

- regression: MAE, MSE, R². classification: accuracy, precision, recall, F1, ROC AUC, confusion matrix.
- **interpretability**: SHAP(tree/ensemble 효과적, NN은 DeepExplainer), ablation study(NN 컴포넌트 제거 평가). SHAP 입력은 transformed feature.
- **bias test**: gender/age/ethnicity 등 slice별 평가. **training_helper_columns**(예: gender)로 feature 아닌 채로 그룹화(training 전 drop).
- model 파일: .safetensors(LLM), .pkl(Scikit, 보안위험), .json(XGBoost), .onnx, .pt/.pth, .engine(TensorRT), .pb/.h5(TF).
- **model card**: 1페이지 개요(intended use, architecture, performance, bias, deployment, interpretability, compliance). registry에 모델과 함께.

---

## Summary (핵심 정리)
- training pipeline은 label·feature 식별, hyperparameter tuning, fit, 평가·compliance 검증을 다룬다.
- data engineering(label 준비·join), ML engineering(GPU·분산·bottleneck 제거) 스킬도 필요하다.
- self-supervised/unsupervised/supervised 데이터를 feature group에서 관리하고, reproducible training data로 모델을 비교한다.
- model evaluation·bias test·model card로 production 신뢰성을 확보한다.
