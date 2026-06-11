# 06. Model Development and Offline Evaluation

## 챕터 개요 (3줄 요약)
- model 선택은 SOTA에 매몰되지 말고 simple model부터, human bias를 피하며 trade-off와 assumption을 고려해 결정한다.
- ensemble(bagging/boosting/stacking), experiment tracking·versioning, distributed training(data/model/pipeline parallelism), AutoML이 model 개발의 핵심 도구다.
- offline evaluation은 baseline과의 비교가 필수이며 perturbation/invariance/directional test, calibration, slice-based evaluation으로 production 전 검증한다.

---

## 1. Evaluating ML Models
> 시간·compute가 유한하므로 전략적 선택 필요. classical ML(collaborative filtering, gradient-boosted tree)은 사라지지 않고 NN과 함께(ensemble, embedding 추출) 사용.
> task 유형별 일반적 접근(text classification: naive Bayes/logistic/RNN/BERT; anomaly detection: kNN/isolation forest/clustering). 성능 metric뿐 아니라 data·compute·latency·interpretability도 고려.

## 2. Six Tips for Model Selection
> ① **avoid SOTA trap**: SOTA는 static dataset 기준일 뿐, 당신 data에서 더 낫거나 빠르거나 싸다는 보장 없음.
> ② **start simple**: 배포 쉬움(pipeline 일관성 검증)·debug 쉬움·baseline 제공.
> ③ **avoid human bias**: 선호 architecture에 더 많은 실험을 쏟으면 불공정 비교 → 동등 setup 비교.
> ④ **now vs later**: learning curve로 더 많은 data 시 성능 변화 예측(tree 지금 vs NN 나중).
> ⑤ **evaluate trade-offs**: FP/FN(지문 인증 vs COVID screening), compute/accuracy, interpretability/performance.
> ⑥ **understand assumptions**: prediction, IID, smoothness, tractability, boundaries(linear), conditional independence(naive Bayes), normal distribution.

## 3. Ensembles
> base learner 여러 개의 majority/average vote. Kaggle 2021 우승 22개 중 20개, SQuAD 2.0 top 20이 ensemble. 배포·유지 복잡해 production엔 덜 선호되나 작은 성능 향상이 큰 이익일 때(ad CTR) 사용.
> 70% 정확도 classifier 3개(무상관)의 majority vote → 78.4%. base learner 간 상관이 적을수록 유리(서로 다른 model 유형 조합).
> **bagging**(bootstrap aggregating): 복원추출 bootstrap마다 학습, variance↓·overfitting 방지(random forest). unstable method 개선, stable(kNN)은 약간 저하.
> **boosting**: weak→strong, 반복마다 오분류 sample 가중↑(GBM, XGBoost, LightGBM).
> **stacking**: base learner 출력을 meta-learner가 결합.

## 4. Experiment Tracking and Versioning
> hyperparameter 하나(lr 0.003 vs 0.002) 차이로 성능 급변 → 재현 위한 정의·artifact 추적 필요.
> **tracking**: loss curve, 성능 metric, sample/prediction/label log, speed, system metric(memory/GPU), parameter·hyperparameter 변화. observability 제공.
> **versioning**: code뿐 아니라 data도 version(DVC). data versioning 어려움(크기, diff 정의 모호, merge conflict, GDPR). aggressive tracking이 reproducibility를 돕지만 보장은 아님(framework/hardware nondeterminism).

## 5. Debugging ML Models
> 어려운 이유: silent failure(코드는 돌지만 예측 틀림), 검증 느림(retrain 필요), cross-functional 복잡성(여러 팀 소유).
> 실패 원인: theoretical constraint, 구현 버그, 나쁜 hyperparameter, data 문제, 나쁜 feature 선택.
> 기법: start simple & 점진적 component 추가, single batch overfit(작은 data로 최소 loss 확인), random seed 고정(재현성).

## 6. Distributed Training
> data가 memory 초과 → out-of-core·parallel 전처리, gradient checkpointing(memory-compute trade-off, 10x 큰 model을 20% 시간 증가로).
> **data parallelism**: data를 여러 machine에 분할, gradient 누적. synchronous SGD(straggler 문제) vs asynchronous SGD(gradient staleness; sparse update 시 문제 작음). 큰 batch size는 일정 지점 후 diminishing returns.
> **model parallelism**: model의 다른 component를 다른 machine에. pipeline parallelism(micro-batch로 분할해 병렬성↑). data와 model parallelism 병용 가능.

```
data parallelism  : same model copy, split data, accumulate gradients
model parallelism : split model components across machines
pipeline          : micro-batches overlap forward/backward passes
```

## 7. AutoML
> **soft AutoML (hyperparameter tuning)**: lr, batch size, layer 수 등 탐색. 약한 model도 잘 튜닝하면 강한 model 능가. random/grid search, Bayesian optimization (auto-sklearn, Keras Tuner, Ray Tune). test split으로 tuning 금지.
> **hard AutoML (NAS, learned optimizer)**: architecture를 hyperparameter로. NAS = search space + performance estimation + search strategy(RL/evolution). learned optimizer(update rule을 NN으로). 비용 커서 소수 기업만(EfficientNet).

## 8. Four Phases of ML Model Development
> ① before ML(heuristic: "ML 100% 향상이면 heuristic이 50%"), ② simplest ML model(logistic/GBT/kNN, framework 검증), ③ optimizing simple models(objective·hyperparameter·feature·ensemble), ④ complex models(simple 한계 도달 후, decay 속도·retrain 빈도 파악).

## 9. Model Offline Evaluation — Baselines
> metric은 baseline 없이 무의미(F1 0.90도 positive 90% task면 random과 동급).
> 5 baseline: ① random(uniform/label distribution), ② simple heuristic, ③ zero rule(최빈 class 항상 예측), ④ human, ⑤ existing solution. "good system"과 "useful system" 구분(인간보다 잘해도 신뢰 못 받으면 무용).

## 10. Evaluation Methods
> **perturbation test**: test split에 noise 추가(COVID cough에 배경음) → noisy data에 강한 model 선택. noise 민감 model은 유지 어렵고 adversarial attack에 취약.
> **invariance test**: 민감 정보(race/gender) 변경이 출력 바꾸면 bias → 입력에서 제외.
> **directional expectation test**: lot size↑ 시 집값↓이면 잘못 학습.
> **model calibration**: 70% 예측이 실제 70% 적중해야 calibrated(Platt scaling). recommender·click 수 예측에 중요.
> **confidence measurement**: sample별 확신도 threshold(불확실 예측은 discard/human loop).
> **slice-based evaluation**: data를 subset으로 나눠 평가. coarse metric만 보면 minority 차별·critical slice(유료 사용자) 놓침. Simpson's paradox(전체 추세가 subgroup과 반대; Berkeley 1973 입학). slice 발견: heuristics, error analysis, slice finder.

```
Simpson's paradox: B > A overall, yet A > B on every subgroup
```

---

## Summary (핵심 정리)
- model 선택은 SOTA 함정·human bias를 피하고 simple model부터 시작하며 trade-off(FP/FN, compute/accuracy)와 model assumption을 이해해 결정한다.
- ensemble(bagging/boosting/stacking)으로 성능을 높이고, experiment tracking·versioning·distributed training(data/model/pipeline parallelism)·AutoML로 개발을 확장한다.
- offline evaluation은 5종 baseline과 비교가 필수이며, perturbation/invariance/directional test·calibration·confidence·slice-based evaluation으로 production 배포 전 robustness·fairness·calibration을 검증한다.
