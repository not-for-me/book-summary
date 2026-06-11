# 08. Data Distribution Shifts and Monitoring

## 챕터 개요 (3줄 요약)
- ML system 실패는 software system failure(non-ML)와 ML-specific failure(production data 차이, edge case, degenerate feedback loop)로 나뉜다.
- data distribution shift는 covariate shift, label shift, concept drift로 구분되며 statistical test로 탐지하고 retraining 등으로 대응한다.
- monitoring(metric 추적)과 observability(내부 상태 추론)로 operational·ML-specific metric을 logs/dashboard/alert로 관리한다.

---

## 1. Causes of ML System Failures
> failure = system 기대(operational + ML performance) 위반. operational 위반은 탐지 쉬움(timeout, 404), ML performance 위반은 어려움(silent failure, 정답 모름).
> **software system failure**: dependency, deployment, hardware, downtime. Google 96 failure 중 60개가 non-ML(distributed system, data pipeline). ML engineering은 대부분 engineering.

## 2. ML-Specific Failures
> **production data ≠ training data**: train/unseen data가 유사 분포라는 가정이 틀림(real-world는 무한·non-stationary). train-serving skew(개발은 좋으나 배포 후 저하), 시간에 따른 점진적 저하. shift는 갑자기/점진적/계절적 발생. monitoring dashboard의 shift 다수는 실제론 internal error(pipeline bug 등).
> **edge case**: 모델이 치명적 실수를 하는 극단 sample(self-driving 0.01%). outlier(data가 다름) vs edge case(성능이 나쁨) — 모든 outlier가 edge case는 아님. inference 시엔 outlier 제거 불가.

## 3. Degenerate Feedback Loops
> system 출력이 미래 입력을 생성해 미래 출력에 영향. natural label task(recommender, ad CTR)에 흔함. A가 약간 높게 랭크→더 클릭→더 높게 랭크(exposure/popularity bias, filter bubble, echo chamber). resume 모델이 feature X("Stanford 출신")를 강화하며 bias 증폭.
> **탐지**: offline에선 어려움. recommender는 popularity diversity(aggregate diversity, long-tail coverage), hit rate vs popularity bucket으로 측정.
> **교정**: ① randomization(TikTok이 신규 영상에 무작위 traffic pool 할당; UX 비용), ② positional feature(추천 위치 정보를 feature로 추가해 위치 영향 학습; inference 시 1st Position=False, 또는 2-model 접근).

## 4. Data Distribution Shifts — Types
> source distribution(train) vs target distribution(inference). 결합분포 P(X,Y)=P(Y|X)P(X)=P(X|Y)P(Y).
> **covariate shift**: P(X) 변화, P(Y|X) 불변(breast cancer: 40세↑ 비율 차이나 given age 발병확률 동일). 원인: selection bias, oversampling, active learning, 환경 변화.
> **label shift**(prior shift): P(Y) 변화, P(X|Y) 불변. covariate shift와 함께 발생 가능(모두는 아님).
> **concept drift**(posterior shift): P(Y|X) 변화, P(X) 불변("same input, different output"; COVID로 SF 집값 변화). 주기적·계절적인 경우 많음.

```
covariate shift: P(X) changes, P(Y|X) same
label shift:     P(Y) changes, P(X|Y) same
concept drift:   P(Y|X) changes, P(X) same
```

## 5. General Shifts & Detecting
> **feature change**(신규/제거/값 범위 변경, age years→months), **label schema change**(Y 가능값 변경; class 추가/세분화 시 relabel+retrain).
> **탐지**: 우선 accuracy-related metric(label 필요, production에선 지연/부재). label 없으면 P(X)·P(Y) 등 분포 monitoring.
> **statistical methods**: summary statistic(min/max/mean/median/variance) 비교는 시작점이나 불충분. two-sample test(KS test — 1D만, 비용·false positive; Least-Squares Density Difference; MMD). 고차원은 차원축소 후 test(Alibi Detect).

## 6. Time Scale Windows & Addressing
> 급격한 shift는 탐지 쉽고 점진적은 어려움. spatial(access point) vs temporal(시간). temporal은 time-series로 처리. time scale window가 탐지 가능 shift를 좌우(주간 cycle은 1주 미만 window로 못 봄). cumulative(과거 포함, 특정 window 은폐) vs sliding statistic. RCA로 변화 시점 자동 분석.
> **대응**: ① 거대 dataset으로 학습, ② label 없이 target 적응(domain-invariant representation; 연구 단계), ③ retraining(stateless from scratch vs stateful fine-tuning; 어떤 data로). feature 선택 시 performance vs stability trade-off(app ranking은 빠르게 변함 → bucket화). 시장별 분리 model로 적응성↑. 많은 failure는 여전히 human error.

## 7. Monitoring vs Observability
> **monitoring**: metric 추적·측정·logging으로 문제 발생 시점 파악. **observability**: 외부 출력으로 내부 상태를 추론하도록 system 구성(instrumentation). observability ⊃ monitoring 전제.
> **operational metric**: network/machine/application 3수준(latency, throughput, CPU/GPU util). availability=uptime, SLO/SLA(AWS EC2 99.99% = 월 ~4분 down).

## 8. ML-Specific Metrics
> 4 artifact(변환 깊을수록 error 가능성↑이나 monitoring 쉬움): accuracy-related, prediction, feature, raw input.
> **accuracy-related**: user feedback(click/완료율) log → natural label. 가장 직접적.
> **prediction**: 저차원이라 monitoring 쉬움, two-sample test로 shift 탐지(입력 shift의 proxy), 이상 패턴(연속 False) 즉시 탐지.
> **feature**: feature validation(schema 준수: min/max 범위, regex, 집합 소속). table testing(Great Expectations, Deequ). 4 우려: 비용(수백 model×수천 feature), 성능 저하 탐지엔 부족, multi-step 추출의 원인 식별 곤란, schema 변경. alert fatigue 주의.
> **raw input**: 다양한 source·format, ML engineer 직접 접근 어려움(data platform 팀 책임).

## 9. Monitoring Toolbox & Observability
> **logs**: runtime event 기록("If it moves, we track it"). distributed tracing(unique ID + metadata). ML로 log 분석(anomaly detection). batch(주기적 발견) vs stream processing(즉시).
> **dashboards**: metric 시각화, 비엔지니어 접근성. 단 graph만으론 부족(통계 지식 필요), dashboard rot 주의.
> **alerts**: alert policy(조건) + notification channel(대상) + description(runbook). alert fatigue로 critical alert 둔감화 → 의미 있는 조건만.
> **observability**: telemetry(remote 측정) 기반, 외부 출력으로 내부 상태 추론. fine-grain(어떤 입력/사용자/기간에 저하) query 가능. ML에선 interpretability 포함.

---

## Summary (핵심 정리)
- ML 실패는 software system failure(non-ML, 현재 다수)와 ML-specific failure(production data 차이, edge case, degenerate feedback loop)로 나뉘며 후자는 탐지·수정이 어렵다.
- data distribution shift는 covariate(P(X))·label(P(Y))·concept drift(P(Y|X))로 구분되고, accuracy metric·분포 monitoring·two-sample test(KS/MMD)로 탐지하며 retraining(stateless/stateful)으로 대응한다.
- monitoring(operational + ML-specific metric: accuracy/prediction/feature/raw input)과 observability(telemetry로 내부 상태 추론)를 logs·dashboard·alert로 운영하되 alert fatigue·dashboard rot를 경계한다.
