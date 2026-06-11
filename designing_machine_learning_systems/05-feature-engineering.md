# 05. Feature Engineering

## 챕터 개요 (3줄 요약)
- 좋은 feature는 알고리즘 기법보다 성능에 더 큰 boost를 주며, deep learning이 일부 feature를 자동 학습해도 feature engineering은 여전히 핵심이다.
- 주요 연산은 missing value 처리, scaling, discretization, categorical encoding(hashing trick), feature crossing, positional embedding이다.
- data leakage는 label이 feature로 새어 들어가 production에서 모델을 실패시키는 위험이며, feature는 importance와 generalization으로 평가한다.

---

## 1. Learned vs Engineered Features
> deep learning(feature learning)은 raw text/image에서 feature를 자동 추출(tokenization, one-hot)하나, 아직 모든 feature 자동화는 멀었고 production 다수는 non-DL이다.
> 텍스트 외 정보(comment의 upvote, user 계정 생성 시점/빈도, thread 조회수)도 필요 → 무엇을 어떻게 추출할지가 feature engineering. 복잡 task는 수백만 feature, domain task는 SME 필요.

## 2. Handling Missing Values
> 3종류: **MNAR**(값 자체 때문에 결측, 고소득자가 소득 미공개), **MAR**(다른 관측 변수 때문, gender A가 age 미공개), **MCAR**(패턴 없음, 드묾).
> **deletion**: column deletion(결측 많은 변수 제거, 중요 정보 손실 위험)·row deletion(MCAR·소량일 때만; MNAR/MAR에선 정보 손실·bias).
> **imputation**: default/mean/median/mode로 채움. 가능한 실제값(예: 자녀 수 0)으로 채우면 결측과 실제값 구분 불가 → 회피. 완벽한 방법 없음(bias·noise·leakage 위험).

## 3. Scaling
> 변수 범위 차이(age 20~40 vs income 10k~150k)를 model이 구분 못함 → [0,1] 또는 [-1,1]로 rescale, 또는 standardization(zero mean, unit variance).
> skewed 분포엔 log transformation. 주의: scaling은 data leakage 원인이며 global statistics(min/max/mean) 필요 → train 통계를 inference에 재사용, 분포 변화 시 retrain.

```
min-max:        x' = (x - min) / (max - min)
standardization: x' = (x - mean) / std
```

## 4. Discretization & Categorical Encoding
> **discretization**(binning/quantization): 연속 feature를 bucket으로(income을 lower/middle/upper). 데이터 적을 때 도움되나 경계 불연속($34,999 vs $35,000) 문제.
> **categorical encoding**: production에서 category는 변함(Amazon 200만+ brand, 신규 brand 계속 등장 → UNKNOWN 처리의 함정).
> **hashing trick**(Vowpal Wabbit): hash function으로 category를 고정 hash space index에 매핑 → 미지 category도 인코딩. collision은 random이라 영향 작음(Booking.com: 50% collision에도 log loss <0.5%↑). continual learning에 유용.

## 5. Feature Crossing
> 둘 이상 feature 결합으로 nonlinear 관계 모델링(marital status × number of children). linear/logistic/tree 모델에 필수, NN엔 덜 중요하나 학습 가속(DeepFM, xDeepFM).
> 단점: feature space 폭증(100×100=10,000) → 더 많은 data 필요, overfitting 위험.

## 6. Positional Embeddings
> Transformer는 word를 병렬 처리 → 순서 명시 입력 필요("dog bites child" ≠ "child bites dog"). 절대 위치(0~7)나 단순 rescale은 NN 학습에 부적합.
> **learned**: word embedding처럼 position embedding matrix(weight 갱신으로 변함). **fixed**: sine/cosine 함수로 사전 정의(Transformer 원논문, even index=sin, odd=cos).
> fixed는 Fourier features의 특수 경우 → 연속 위치(3D 좌표)에도 적용 가능, 좌표 입력 task 성능 향상.

## 7. Data Leakage
> label의 형태가 feature로 "leak"되어 inference 시엔 없는 정보로 예측(COVID 모델이 환자 자세·font를 위험 예측자로 학습; 폐암 모델이 scan 기계 종류에 의존).
> 비명시적이라 위험: 광범위 평가 후에도 production에서 실패. 경험 연구자·Kaggle competition(Ion Switching test label 역설계)에서도 발생.

## 8. Common Causes & Detecting Data Leakage
> 원인: ① time-correlated data를 random split(미래 정보 leak → 시간순 split), ② split 전 scaling(test 통계 leak → split 후 train 통계로 scaling), ③ test split 통계로 missing 채움, ④ split 전 duplicate 미제거(CIFAR 3.3~10% 중복), ⑤ group leakage(상관 sample이 train/test 분산), ⑥ data 생성 과정 leak.
> 탐지: feature별 예측력/상관 측정(비정상 고상관 조사), ablation study, 신규 feature의 급격한 성능 향상 경계, test split 사용 최소화.

```
time-correlated:  split by time, not randomly
scaling/missing:  split first, then use train stats only
duplicates:       dedup before split; oversample after split
```

## 9. Engineering Good Features
> feature가 많을수록 leakage 기회·overfitting·memory·inference latency·technical debt 증가. 무용 feature는 L1 regularization이 이론상 weight를 0으로 줄이나, 실제론 제거가 학습 가속.
> **feature importance**: feature 제거 시 성능 저하 정도로 측정. XGBoost 내장, model-agnostic SHAP(전체+개별 prediction 기여, interpretability). Facebook: top 10 feature가 importance 절반, 마지막 300개는 <1%.
> **feature generalization**: unseen data 일반화. coverage(결측 적을수록↑)와 feature value 분포(train/test 겹침). DAY_OF_THE_WEEK(train 월~토, test 일요일)는 일반화 실패 vs HOUR_OF_THE_DAY는 양호. generalization-specificity trade-off(IS_RUSH_HOUR).

---

## Summary (핵심 정리)
- ML 성공은 여전히 feature에 의존하므로 feature engineering에 투자해야 하며, missing value(MNAR/MAR/MCAR)·scaling·discretization·hashing trick·feature crossing·positional embedding이 핵심 연산이다.
- data leakage(label이 feature로 새어듦)는 비명시적이고 치명적이며, time 기준 split·split 후 scaling·train 통계만 사용·dedup 등으로 예방하고 ablation study로 탐지한다.
- feature는 importance(SHAP, 제거 시 성능 저하)와 generalization(coverage, train/test value 분포 겹침)으로 평가하며, 무용 feature는 제거하고 data lineage를 추적하는 것이 best practice다.
