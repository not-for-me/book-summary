# 07. Model-Dependent and On-Demand Transformations

## 챕터 개요 (3줄 요약)
- MDT는 feature store에서 읽은 뒤 모델별로 적용되는 변환(encoding, scaling, tokenization)으로 training·inference 양쪽에서 동일하게 실행해 skew를 막아야 한다.
- ODT는 online inference·feature pipeline에서 실행되는 보통 stateless 변환으로, training data 통계를 파라미터로 갖지 않는다.
- Scikit-Learn pipeline, Hopsworks feature view transformation, PyTorch transform으로 skew 없이 구현하고 pytest로 unit test한다.

---

## 1. Feature Transformations
> categorical은 encoding, numerical은 scaling/distribution 변환. training data 통계로 parameterize된다.

### Categorical encoding
- **one-hot**(낮은 cardinality, ordinal 없음), **ordinal**(순서 보존), **feature hasher**(고cardinality, hash collision 위험), **label encoder**(target). 새 category 대비 "unknown/other" 포함.
- CatBoost는 categorical 인코딩 불필요(고cardinality·ordinal 자동 처리).

### Numerical distributions & transforms
- 분포 파악(normal/uniform/binomial/Poisson/exponential/skewed/bimodal/log-normal).
- **standardization**(mean 0, std 1; gradient descent·kNN·SVM·linear) vs **normalization/min-max**(범위 0~1, 분포 형태 보존; neural net·outlier 의미 있을 때).
- **log transform**(right-skew 완화, 0/음수 불가→log(1+x)). tree 모델은 scale 불필요(단 skew 완화는 도움).
- 공식: min-max, z-score, log, reciprocal, exponential, Box-Cox(λ=0이면 ln).
- 변환은 2-pass(통계 계산 → 적용).

### 저장
- 변환 feature는 보통 feature group에 저장 안 함(재사용 불가, write amplification). 단 최저 지연 real-time은 online-only "transformed" feature group + 전용 pipeline 가능.

---

## 2. Model-Specific Transformations
> feature transformation 외 단일 모델 특화 변환(imputation, outlier, label-dependent, tokenization 등).

- **outlier handling**: 가능하면 ingest 전 제거(Great Expectations). 아니면 MDT(z-score/IQR, Isolation Forest/LOF). log 전에 outlier 제거.
- **imputing missing**: time-series는 forward/backward fill(Pandas ffill/bfill, PySpark window function), non-time-series는 SimpleImputer(mean/median/mode) 또는 feature view 변환(대용량). model-based(IterativeImputer).
- **model-based cleaning**: training pipeline에서만(inference엔 unclean data). 예: Llama 3.1 text-quality classifier(Llama 2가 학습 데이터 생성). Cleanlab(label error), Lightly(image).
- **target/label-dependent**: label timestamp로 parameterize(예: time_since_last_transaction). 필요 시점에만 계산.
- **expensive features**: 모든 entity precompute 낭비면 MDT로(필요 시만). 모든 feature가 MDT면 feature pipeline 제거 가능.
- **tokenizer/chat template**: tokenization은 LLM 버전 결합 MDT(Llama 3 토큰을 Llama 2/4에 못 씀). HF chat template로 tokenizer-모델 결합해 skew 방지. (text chunking은 MIT, tokenization과 분리해야 재사용 가능.)

---

## 3. Transformations in Scikit-Learn Pipelines
> transformer + model을 Pipeline 객체로 묶어 pickle 저장 → inference에서 동일 적용(skew 방지).

```python
preprocessor = ColumnTransformer([
  ("num", Pipeline([("imputer",SimpleImputer("median")),("scaler",StandardScaler())]), numerical),
  ("cat", Pipeline([("encoder",OneHotEncoder(handle_unknown="ignore"))]), categorical)])
clf = Pipeline([("preprocessor",preprocessor),("classifier",LogisticRegression())])
clf.fit(X_train, y_train); joblib.dump(clf, "cc_fraud.pkl")
```

- inference: pipeline 객체 다운로드 → `clf.predict(df)`가 변환+예측 한 번에. joblib 버전 일치 주의.
- transformer: SimpleImputer/IterativeImputer, OneHotEncoder/OrdinalEncoder/TargetEncoder, StandardScaler/MinMaxScaler/MaxAbsScaler, quantile/power(Box-Cox/Yeo-Johnson), normalize(l1/l2/max). 단 NumPy 기반(Arrow-backed DataFrame 미지원).

---

## 4. Transformations in Feature Views (Hopsworks)
> feature view에 변환 함수를 선언적으로 부착해 read 후 client에서 실행, skew 없이 MDT 적용. Arrow-backed.

- 내장(one_hot_encoder, min_max_scaler, label_encoder) + custom UDF. training data 통계는 training dataset 객체(model registry)에 저장.
- `train_test_split()`/`get_batch_data()`/`get_feature_vector()` 기본 transformed=True. 변환이 컬럼 수·순서를 바꿔도 일관 보장. 개발자는 feature view·string 컬럼만 다룸(model signature는 feature view가 매핑).
- **mixed-mode UDF**: Python(저지연 online) / Pandas(대용량) 양쪽 실행. `@hopsworks.udf`.

```python
stats = TransformationStatistics("amount")
@hopsworks.udf(float)
def transaction_amount_deviation(amount, statistics=stats):
    return amount / statistics.amount.mean   # 통계 사용 → MDT
```

- MIT vs MDT vs ODT 선택(예: days_to_card_expiry): batch면 재사용·pipeline overhead로 판단, real-time면 precompute 여부로 판단. 통계 파라미터 있으면 MDT.

---

## 5. On-Demand Transformations
> training data 통계 없는 변환 함수를 ODT로 사용. feature group에 등록(feature pipeline에서도 실행).

- request-time param + (inference helper로 읽은) precomputed feature 조합. MDT와 달리 **feature group에 등록**(feature pipeline 실행 가능).
- `fg.insert(df)` 시 ODT 실행. 파라미터명이 컬럼명과 일치해야. `drop=[...]`로 비-feature 컬럼 제거.

---

## 6. PyTorch Transformations
> unstructured 데이터를 tensor로 변환. 이미지 전처리를 feature pipeline(ODT/augmentation)로 옮겨 GPU util을 높인다.

- 예: celebrity twin(ResNet + CelebA). FTI로 image transform을 training→feature pipeline 이동 → training CPU 부하↓, GPU util↑.
- Torchvision v2 `Compose`로 ODT(Resize/CenterCrop), MIT(+ImageAugmentation: flip/color/erase로 overfit 방지), MDT(ToImage/ToDtype/Normalize) 분리. feature pipeline은 PNG 출력, training/inference에서 PNG→tensor(MDT).

---

## 7. Using pytest
> feature/transformation 함수를 unit test해 의도치 않은 변경이 client를 깨뜨리지 않게 한다.

- 예: days_to_card_expiry 변경(max(days,1))이 log transform과 충돌해 모델 성능 저하 → unit test가 사전 포착.
- test 명명: `Test*` 클래스, `test_*` 함수. **arrange-act-assert** 패턴. invariant/precondition/postcondition 검증.
- LLM으로 unit test·edge case 생성하되 검증 필수(CrowdStrike 2024 사례: null 미검사 + 자동 테스트 부재).
- 실행: `python -m pytest`(개발/CI만, production 불필요). GitHub Action으로 push/PR 시 자동 실행.
- 방법론: 모든 feature/transformation 함수 unit test + coverage, pipeline e2e test, utility test. edge case·LLM 활용. 실험 단계엔 test-first 비권장.

---

## Summary (핵심 정리)
- MDT·ODT를 데이터과학·엔지니어링 관점에서 다뤘다(categorical·numerical 변환).
- skew 없는 MDT 구현 프레임워크: Scikit-Learn pipeline(소규모), Hopsworks feature view(Pandas UDF 대규모, MDT·ODT 모두).
- PyTorch 예제로 FTI에서 image/tensor 변환(MIT/MDT/ODT)을 조직했다.
- pytest로 transformation 함수를 unit test하는 법을 소개했다.
