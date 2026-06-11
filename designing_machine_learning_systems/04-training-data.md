# 04. Training Data

## 챕터 개요 (3줄 요약)
- training data는 nonprobability·random sampling으로 선택하며, 각 방법의 sampling bias를 이해하고 적절히 골라야 한다.
- labeling은 hand label, natural label(feedback loop length), 그리고 label 부족 대응책(weak/semi-supervision, transfer/active learning)으로 나뉜다.
- class imbalance는 현실의 norm이며 올바른 metric·resampling·loss 조정으로 다루고, data augmentation으로 data 양과 robustness를 높인다.

---

## 1. Sampling — Nonprobability
> 확률 기준 없는 선택 → selection bias. convenience(가용성), snowball(기존 sample 기반 확장), judgment(전문가 선정), quota(slice별 할당, 무작위 없음).
> 실제로 language model(Wikipedia/Reddit), sentiment(IMDB/Amazon review), self-driving(맑은 날씨 편중) 등 편의적 선택이 흔함. 초기 data 확보엔 빠르나 신뢰 모델엔 부적합.

## 2. Sampling — Random
> **simple random**: 모두 동일 확률, 구현 쉬우나 rare class(0.01%)가 누락될 수 있음.
> **stratified**: population을 stratum으로 나눠 각 group에서 sampling → rare class 보장. multilabel처럼 group 분할 불가 시 어려움.
> **weighted**: sample별 weight로 선택 확률 조정 → domain expertise 반영, 실제 분포와 다른 data 보정. (sample weight는 loss 영향도 조정으로 weighted sampling과 구분.)
> **reservoir sampling**: streaming data에서 크기 모를 때 k개를 동일 확률로 추출(reservoir 배열, n번째 원소가 k/n 확률). 언제 멈춰도 올바른 확률.
> **importance sampling**: P(x) 추출이 어려울 때 쉬운 Q(x)에서 뽑고 P(x)/Q(x)로 가중. policy-based RL에서 old policy로 reweight.

## 3. Labeling — Hand Labels
> 단점: 비싸고(전문가 필요 시), privacy 위협(데이터 외부 반출 불가), 느림(음성 phonetic 전사는 발화의 400배 시간) → iteration 느려 환경 변화 적응 저하.
> **label multiplicity**: 여러 source·annotator의 충돌 label(entity recognition 예). 명확한 problem definition과 annotator 교육으로 disagreement 최소화.
> **data lineage**: 각 sample·label의 출처 추적 → bias 탐지·debugging(저품질 신규 data가 성능 저하 원인일 때 식별).

## 4. Natural Labels
> system이 prediction을 자동/부분 평가(Google Maps ETA, 주가 예측, recommender의 click=behavioral label). 86개 기업 조사 63%가 natural label 작업(시작이 쉽고 저렴).
> implicit label(positive 부재로 negative 추정) vs explicit label(직접 평가).
> **feedback loop length**: prediction~feedback 도착 시간. short(추천 click 수분) vs long(fraud detection 1~3개월 dispute window). user feedback 종류(click vs purchase)는 volume·신호 강도·loop length가 다름. window 길이는 speed-accuracy trade-off(짧으면 premature negative label).

## 5. Handling the Lack of Labels
> **weak supervision**(Snorkel): labeling function(LF, heuristic 인코딩: keyword/regex/DB lookup/타 model 출력)으로 noisy label 생성. ground truth 불필요(소량 권장). programmatic labeling은 hand 대비 저렴·privacy·빠름·adaptive. Stanford 사례: 8시간 LF 작성이 1년 hand labeling과 동등 성능.
> **semi-supervision**: 소량 초기 label + 구조적 가정으로 label 확장. self-training(고확률 예측 추가), 유사 sample 동일 label 가정, perturbation 기반.
> **transfer learning**: base task(예: language modeling)로 학습한 model을 downstream task에 재사용. zero-shot 또는 fine-tuning. label 적은 task에 특히 유용, 대형 pretrained model일수록 성능↑(GPT-3 훈련비 수천만 달러).
> **active learning**(query learning): model이 가장 유용한 sample 선택 라벨링. uncertainty measurement, query-by-committee(model 위원회 불일치), gradient/loss 기반.

```
weak supervision  : noisy heuristics (LF)
semi-supervision  : structural assumptions + seed labels
transfer learning : pretrained base model -> downstream
active learning   : label the most useful samples
```

## 6. Class Imbalance — Challenges
> class 간 sample 수 큰 차이(폐암 X-ray 99.99% normal). regression에서도 발생(health-care bill의 95th percentile).
> 학습 곤란 3이유: ① minority 신호 부족(few-shot), ② 단순 heuristic(항상 majority 출력 시 99.99% accuracy)에 갇힘, ③ 비대칭 오류 비용(cancer 오분류가 더 위험).
> 원인: 본질적 희소(fraud, churn, disease/resume screening, object detection), sampling bias, labeling error.

## 7. Handling Class Imbalance — Metrics & Data-level
> **right metrics**: accuracy/error rate는 majority 지배 → 부적절. class별 accuracy, precision/recall/F1(positive class 기준, 비대칭), ROC-AUC, 심한 imbalance엔 Precision-Recall curve 권장.
> **resampling**: oversampling(minority 복제, overfitting 위험)·undersampling(majority 제거, data 손실 위험). Tomek links(경계 명확화), SMOTE(minority convex 결합 합성) — 저차원에서만 입증. two-phase learning, dynamic sampling. resampled data로 evaluate 금지.

```
Precision = TP / (TP + FP)
Recall    = TP / (TP + FN)
F1        = 2 * P * R / (P + R)
```

## 8. Handling Class Imbalance — Algorithm-level
> loss function 조정으로 robust화. **cost-sensitive learning**(Elkan): cost matrix Cij로 오분류 비용 반영(수동 정의 필요).
> **class-balanced loss**: class weight를 sample 수에 반비례 → rare class 가중.
> **focal loss**: 맞히기 어려운(저확률) sample에 높은 weight → 어려운 sample에 집중. ensemble도 imbalance에 도움(Ch6).

## 9. Data Augmentation
> **simple label-preserving**: vision은 crop/flip/rotate/erase(회전한 개도 개), NLP는 유사어 치환. data 2~3배 손쉽게 증가.
> **perturbation**(noise 추가): NN은 noise에 민감(1 pixel 변경으로 오분류) → adversarial attack. noisy sample 추가(DeepFool, adversarial augmentation)로 robust화. BERT는 token 15% 중 일부를 random word 치환.
> **data synthesis**: NLP template로 query 대량 생성, vision은 mixup(x'=γx1+(1-γ)x2, label도 결합)으로 generalization↑·robustness↑. GAN(CycleGAN) 합성도 연구 중.

---

## Summary (핵심 정리)
- training data가 ML의 토대이며, nonprobability(편의·bias)와 random(simple/stratified/weighted/reservoir/importance) sampling을 문제에 맞게 선택해 sampling bias를 피해야 한다.
- 대부분 supervised이므로 labeling이 핵심이고, natural label(feedback loop length)·hand label의 한계를 weak/semi-supervision, transfer/active learning으로 보완한다.
- class imbalance는 현실의 norm으로 올바른 metric(precision/recall/F1/PR curve), data-level resampling(SMOTE/Tomek), algorithm-level loss 조정(cost-sensitive/class-balanced/focal loss)으로 다루며, data augmentation으로 양과 robustness를 함께 높인다.
