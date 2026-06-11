# 04. Automating Data Quality Monitoring with Machine Learning

## 챕터 개요 (3줄 요약)
- data quality monitoring model은 sensitivity, specificity, transparency, scalability 네 요건을 충족해야 하며, outlier detection과는 본질적으로 다른 문제다.
- 핵심 알고리즘: 매일 data snapshot을 떠서 "이 record가 today의 것인가"를 예측하는 classifier를 학습 — 학습이 성공하면 today data에 구조적 변화가 있다는 의미다.
- data sampling, feature encoding, gradient-boosted tree(XGBoost) model 개발, SHAP 기반 explainability로 anomaly score를 산출한다.

---

## 1. Requirements
> data quality monitoring model은 sensitivity, specificity, transparency, scalability를 갖춰야 한다.

- **Sensitivity**: true positive 탐지력. 현실적 benchmark는 **record의 1% 이상**에 영향을 주는 변화 탐지. 1% 미만은 너무 noisy — 더 작은 변화는 validation rule이나 중요 record subset view에 ML 적용으로 처리.
- **Specificity**: false positive 회피력. alert fatigue 방지에 특히 중요. over-alert 원인: seasonality, correlated column clustering 실패, 너무 작은 sample/time window, dataset별 chaos 수준 차이.
- **Transparency**: issue 발생 시 사용자가 이해·root cause하도록 도와야 함. model architecture가 설명·attribution 가능 범위를 결정.
- **Scalability**: 매일 billions row에 대해 human/storage/compute 비용으로 실행 가능해야 함. up-front config·retuning 불필요(그게 곧 handwritten rule), warehouse query footprint 최소화, 저렴한 외부 hardware에서 빠른 실행.

### 1.1 Nonrequirements
> 무엇을 안 해도 되는지 정의하는 것도 똑같이 유용하다 (four-pillar 중 한 축일 뿐).

- 개별 bad record 식별 불필요(그건 rule의 역할), real-time 처리 불필요(daily/hourly batch), 항상 corrupted였던 data 탐지 불가(ML은 historical로 학습), time 개념 없는 data 분석 불가(timestamp 필요).

### 1.2 Data Quality Monitoring Is Not Outlier Detection
> outlier detection(예: Isolation Forest)은 분포의 중심에서 먼 row를 찾을 뿐, 분포의 갑작스런 구조적 변화를 찾는 monitoring과 근본적으로 다른 문제다.

- 모든 dataset엔 outlier가 존재(정규분포도 극값 있음). outlier가 곧 quality issue는 아님. monitoring은 "과거의 분포·pattern·관계가 지금 갑자기 유의하게 바뀌었는가"를 본다.

---

## 2. ML Approach and Algorithm
> 핵심 통찰: 매일 snapshot을 뜨고 "data가 today의 것인지" 예측하는 classifier를 학습 — 학습 불가(동전 던지기)면 정상, 예측 가능하면 today data가 비정상이다.

- human label 없는 unsupervised task지만, data 도착 시점(date)을 label처럼 사용 → unsupervised/semisupervised 논쟁 가능(Ch5에서 labeled anomaly로 fine-tune).
- 변화가 significant해도 흥미롭지 않을 수 있음(date column은 매일 변함 → Ch5에서 처리; 사용자가 관심 없는 변화 → Ch6에서 처리).
- 단계: data sampling → feature encoding → model development → model explainability.

### 2.1 Data Sampling
> today(label=1)와 not-today(label=0, 여러 lookback date 혼합)에서 random sample을 구성한다.

- not-today는 yesterday(급변 탐지) + 1주 전(요일 seasonality) + 2주 전(지난주 anomaly 대비) 등 혼합.
- **Shadow anomalies**: 과거 anomaly 있던 date와 비교하면 가짜 변화로 보임(지난주 Android 결측 → 오늘 Android 급증처럼). 여러 lookback date sampling·강한 anomaly date 제외로 방지.
- **Sample size**: 하루 최소 100 record, **10,000 record면 1~5% 영향 issue까지 대부분 포착**. 100,000 초과 시 개선 미미. 정확도는 전체 크기가 아닌 **절대 sample 크기**에 의존(중국 vs 룩셈부르크 평균소득 추정 비유).
- **Bias and efficiency**: sampling에 bias 있으면 ML이 그걸 false positive로 탐지. random sampling이 가장 비싼 연산일 수 있음. date column으로 partition, `TABLESAMPLE`로 필요량보다 약간 크게 추출 후 `random() <= X`로 distributed 최종 sampling(`order by random() limit`은 비효율). WHERE는 `created_date = '2023-06-01'`처럼 partition metadata 활용(cast 금지). 단 BigQuery TABLESAMPLE은 block 단위라 bias 위험.

### 2.2 Feature Encoding
> 자동화된 feature engineering으로 각 column을 float matrix로 encoding한다.

- 추천 encoder: **Numeric**(bool/int/float→float), **Frequency**(값 등장 빈도), **IsNull**(NULL 여부), **TimeDelta**(생성 시각과의 초 차이), **SecondOfDay**(생성 시각), **OneHot**.
- TF-IDF·mean encoding·PCA·embedding 등은 tree model에 부적합하거나 해석 어려움. encoder가 복잡하면 사용자에게 issue 설명이 어려워짐(예: "gap" encoder는 무관한 변화까지 탐지).

### 2.3 Model Development
> gradient-boosted decision tree(XGBoost)가 빠른 학습·추론, 작은 sample, 모든 tabular data 일반화에 적합하다.

- 이전 tree들의 mistake를 보정하는 tree를 순차 추가하는 ensemble. tuning parameter 적음(learning rate, tree complexity). linear는 너무 단순, neural network는 너무 복잡(대량 이질 data 필요).
- 단점: feature engineering 필요. "꼭 ML이어야 하나?" → column 간 nonlinear 관계·correlation 통제를 위해 multivariate model 필요하고, SHAP로 column 비교만큼 명확히 설명 가능.
- **Training/evaluation**: holdout set으로 매 step 평가, tree 수 상한 설정. 세 pattern — **No anomaly**(test error 빨리 악화), **Incomplete**(미수렴 → tree·learning rate 증가), **Optimal**(test loss 최소점에서 stop). 단일 dataset 최적화와 일반화 사이 균형 필요.
- **Computational efficiency**: sample 고정으로 비용은 column 수에 linear(JSON 확장 시 10,000 column 가능). 하루치만 query·snapshot(단 cold start 약화), efficient random sampling, tree depth·수 제한·early stop, sparse encoding·multiprocessing·GPU 활용.

### 2.4 Model Explainability
> SHAP value로 각 {row, column} cell이 예측에 기여한 정도를 점수화해 anomaly의 위치·심각도를 설명한다.

- 왜 중요한가: ① 오늘 data가 얼마나 anomalous한지(tuning·우선순위), ② anomaly가 data의 어디에 있는지(root cause·bad data sample).
- 예: credit card data에서 predicted Pr(Today)를 log odds로 변환 후 SHAP로 column별 기여 분해 → FICO=578, BRAND='Mastercard'가 크게 기여 → Mastercard 저신용 분포 이상 시사(실제론 10,000 record 전체로 집계).
- 정규화·tuning 후 **anomaly score** 산출 — cell/row/segment/column/table 단위로 집계·slice 가능, correlation clustering도 가능.
- score를 6 bucket으로 분류(2 bucket마다 한 자릿수 차이): **Minimal, Weak, Moderate, Strong, Severe, Extreme**. table마다 변동성이 달라 custom threshold 학습 필요(Ch5).

---

## 3. Putting It Together with Pseudocode
> 두 날짜 sample을 query→binary label 부여→feature encoding→XGBoost 학습(early stopping)→SHAP→column anomaly score 반환하는 흐름이다.

```python
def detect_anomalies(table, time_column, current_date, prior_date, sample_size):
    data_current = query_random_sample(table, time_column, current_date, sample_size)
    data_prior   = query_random_sample(table, time_column, prior_date, sample_size)
    y = [1]*len(data_current) + [0]*len(data_prior)
    data_all = pd.concat([data_current, data_prior], ignore_index=True)
    # determine + encode features -> X
    X_train, X_eval, y_train, y_eval = train_test_split(X, y, test_size=0.2)
    model = xgb.XGBClassifier()
    model.fit(X_train, y_train, early_stopping_rounds=10, eval_set=[(X_eval, y_eval)])
    shap_values = TreeExplainer(model).shap_values(X)
    return compute_column_scores(shap_values, feature_list)
```
- seasonality·multiple lookback·correlated feature 등은 단순화를 위해 생략된 개념적 예시.

---

## 4. Other Applications
> 같은 알고리즘으로 legacy issue 탐지와 두 dataset 비교가 가능하다.

- **Legacy issue(backtesting)**: 과거 date 시퀀스에 알고리즘 실행해 history의 shock·scar 탐지(Ch5의 model 효과 측정에 사용). 단 설명 불가한 변화나, 이미 backfill로 복구돼 탐지 안 되는 경우 주의.
- **두 sample 비교**: 같은/다른 table의 동일 schema 두 sample 차이를 탐지·설명(sampling 기반이라 거대 table·다른 warehouse도 가능). 활용: source DB vs 변환된 warehouse, 기존 vs 신규 ETL pipeline, 현재 vs 과거, 사업 segment·지역·카테고리 간 비교.

---

## Summary (핵심 정리)
- model은 sensitivity(1%+ 변화), specificity(alert fatigue 회피), transparency, scalability를 충족해야 한다.
- data quality monitoring은 분포의 구조적 변화를 찾는 것으로, outlier detection과 다르다.
- 핵심은 "today vs not-today" classifier 학습 — 예측이 가능하면 변화가 있다는 신호다.
- random sampling(10,000 row), 자동 feature encoding, XGBoost, SHAP anomaly score가 4대 구성요소다.
- 같은 접근으로 backtesting(legacy issue)과 두 dataset 비교까지 확장된다. 단 seasonality·correlation 등 현실 난제는 Ch5에서 다룬다.
