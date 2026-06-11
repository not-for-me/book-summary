# 14. Observability and Monitoring AI Systems

## 챕터 개요 (3줄 요약)
- observability는 metrics(성능 측정)와 logs(내부 상태·trace)의 두 기둥 위에 서며, AI system 운영 문제를 탐지·진단한다.
- ML model은 logs로 feature drift·concept drift를 모니터링하고, outcome 없으면 DLE/CBPE로 성능을 추정한다.
- agent는 logs/traces로 error analysis·eval을 수행하고, guardrail로 유해 입출력을, token 기반 metric으로 LLM을 모니터링한다.

---

## 1. Logging and Metrics for ML Models
> inference pipeline이 metrics(Prometheus)와 logs(테이블)를 출력. 예측 관련 로그는 단일 테이블로 통합.

- model 입출력뿐 아니라 **untransformed feature**(feature monitoring용)·**transformed feature**(model monitoring/SHAP)·request 모두 로깅.
- 로그 저장: lakehouse(batch), online feature group+TTL(real-time), document store(OpenSearch), 관계형, SaaS.
- **blocking(SaaS, 네트워크 지연·손실) vs nonblocking(별도 스레드, 저지연·robust)**. Hopsworks는 feature store 로깅.
- Hopsworks 통합 로그(Table 14-1): model metadata, untransformed/transformed feature, inference helper, predictions. online은 serving key·ODT param 추가.
- `model_mr.log(inference_data)` 또는 `fv.log(df)`. `online=True`면 online feature group(TTL).
- 비교: Databricks(AI Gateway inference table, untransformed 미저장), SageMaker(data capture→S3, CloudWatch).

### Metrics for Online Models
- p99 latency 등 SLO 판단 → autoscaling. host/container metric + application metric(req/sec).
- Prometheus가 `/metrics` scrape → Prometheus Adapter/KEDA가 horizontal pod autoscaler 구동. KServe YAML(minReplicas/maxReplicas/trigger).
- **scale-to-zero**: 비용↓하나 cold-start(decision tree 10-20초, LLM 수분).

### Metrics for Batch Models
- batch job autoscaling = 리소스 변경 후 재시작(예: executor OOM→메모리 증가). LinkedIn right-sizer(Kafka→Samza→MySQL→정책). SparkMeasure, LLM으로 right-sizing.

---

## 2. Monitoring Features and Models
> drift = feature/label/관계의 분포 변화로 성능 저하. reference vs detection 분포 비교.

- nonstationary 데이터로 학습한 모델은 시간이 지나며 성능 저하(예: 새 fraud 수법) → 재학습/재설계.
- 분포: features N(X)/F(X)/P(X)/I(X), labels N(y)/F(y)/P(y), predictions Q(ŷ), outcomes Q(y).
- **drift 유형**:
  - **data ingestion drift**: N(X) vs F(X)(나쁜 데이터 조기 경고).
  - **feature drift**: I(X) vs P(X)(inference vs training, 문제일 수도 아닐 수도).
  - **concept drift**: feature-label 관계 변화. 분포 아닌 outcome vs prediction(ROC AUC/MSE).
  - **prediction drift**: Q(ŷ) vs P(y)이나 feature drift 없음.
  - **label shift**, **KPI degradation**.
- **drift 탐지 2방법**: ① 통계 hypothesis testing(reference vs detection 분포 비교). ② **model-based**(reference=True/detection=False로 binary classifier 학습, ROC AUC>>0.5면 drift).

### 유형별
- **data ingestion drift**: feature group 일부 reference vs 최근 ingest. eager/lazy. abrupt/incremental/recurring. Hopsworks `create_feature_monitoring`(detection/reference window + compare_on + cron).
- **univariate feature drift**: training(reference) vs inference log(detection). KL divergence/Wasserstein/L-infinity/KS/deviation-from-mean(Gaussian 가정, 효율적).
- **multivariate feature drift**: joint 분포 변화. **NannyML**: PCA reconstruction(variance 기반, 저복잡·해석가능·대규모) vs domain classifier(모든 drift 민감, 복잡 패턴).
- **vector embedding**: 해석 어려움 → downstream task 성능 모니터링 또는 주기적 재계산. domain classifier 권장.

### Model Monitoring (NannyML)
- outcome 있으면 concept drift 직접(prediction vs outcome). 없으면 KPI proxy 또는 model-based:
  - **CBPE**(classification, predict_proba 필요, accuracy/precision/recall 추정, 추가 학습 불필요, feature drift엔 정확하나 concept drift엔 무효).
  - **DLE**(classification+regression, nanny model 학습 필요, loss 직접 추정).
  - reference는 training set 아닌 test set/production(bias 방지). event_time 필요.
- **재학습 vs 재설계**: outcome 있으면 concept drift, 없으면 DLE/CBPE. 많은 feature는 multivariate, 적으면 univariate. 처음엔 alert로 사람 확인, 잦은 alert 후 자동 재학습 schedule. concept/feature drift가 새 데이터 필요 시사하면 재설계(feature/architecture).

---

## 3. Logging and Metrics for Agents
> agent 수준 로깅(다단계). drift 아닌 error analysis·성능 디버깅.

- LLM은 drift 모니터링 안 함(언어·세계는 안정적, 재학습 불가). agent는 error analysis로 prompt/guardrail/RAG/tool/workflow 개선.
- (참고) LRM의 thinking step은 explainability 도구로 신뢰 불가(answer 먼저, 정당화 나중).

### Logs to Traces
- **trace**(trace_id) = span 계층(LLM req/res, RAG, MCP tool). Opik 등으로 로깅(feedback score 포함). Hopsworks 로깅 feature group 저장.

### Error Analysis
> LLM 실수 유형·원인 연구로 성능·신뢰성 개선.

- task별: subjective(instruction-following/harmful/style/factuality/format), objective(reasoning/instruction-following/context-faithfulness/factuality).
- 3단계: trace 분석·annotate → 분류(LLM-as-judge) → 개선(prompt engineering: instruction/example, RAG, LLM 교체, step 변경).
- **log viewer + feedback**(vibe code), LLM-as-judge(같은 LLM도 다른 분류 task면 가능). feedback은 같은/joined feature group.
- **eval 큐레이션**: edge case를 eval로. **algorithmic eval**(query/response만, unit test) vs **verifiable eval**(외부 실행: 코드 컴파일/SQL/test). GitHub Copilot: human-generated criteria 체크박스.

### Guardrails
> 유해 입출력 가능성을 줄이는 input/output detector(helper LLM).

- input: ALLOW/BLOCK/SANITIZE(hate/self-harm/violence/PII/jailbreak). output: 형식 불량·hallucination·정보 유출·toxic.
- 단점: 지연 추가 → 작은 fine-tuned LLM. A/B test에도 사용(Copilot guardrail metric).
- **jailbreaking**(LLM 안전필터 우회, roleplay) vs **prompt injection**(LLM 위에 만든 앱/MCP tool 공격, untrusted input + trusted prompt concat).

### LLM Metrics
- req/sec·latency는 부적합(query/response 길이 편차) → **token throughput·평균 token latency·GPU util**. autoscaling은 token throughput threshold(KServe+vLLM+KEDA). LLM scale-out은 분 단위(GPU 할당 + 디스크 로딩).

---

## Summary (핵심 정리)
- observability·monitoring의 시작은 model 로그 수집(ML vs LLM 상이).
- ML: feature drift·concept drift 모니터링, outcome 없으면 DLE/CBPE.
- univariate·multivariate feature monitoring 병행.
- LLM: 로그로 error analysis·eval 생성. objective는 평가 쉬움, subjective는 LLM-as-judge.
- metric(ML latency, LLM token throughput)으로 bottleneck 식별·autoscaling.
