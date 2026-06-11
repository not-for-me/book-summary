# 02. Introduction to Machine Learning Systems Design

## 챕터 개요 (3줄 요약)
- ML project는 business objective에서 출발해 ML objective로 번역되어야 하며, ML metric은 business metric을 움직일 때만 의미가 있다.
- 좋은 ML system은 reliability, scalability, maintainability, adaptability 4가지 requirement를 만족해야 하고, 개발은 선형이 아니라 iterative cycle이다.
- ML problem framing(input/output/objective function 정의)과 task type 선택이 난이도를 좌우하며, ML 성공은 결국 data의 quality·quantity에 크게 의존한다.

---

## 1. Business and ML Objectives
> data scientist는 accuracy·F1·latency 같은 ML metric에 집착하지만, 기업은 business metric(profit, 직·간접)을 움직이지 못하면 ML project를 조기 중단한다.
> ML 성능을 business 성능에 연결하는 것이 핵심(예: Netflix의 take-rate = quality play / 노출 추천 수 → streaming hours↑·cancellation↓ 연관).
> ML/business metric 관계는 A/B testing 같은 experiment로 검증하며, 복잡한 pipeline에선 ML component의 기여를 분리하기 어려울 수 있다.
> ROI는 adoption 성숙도에 비례(5년 이상 운영 기업의 75%가 30일 내 배포 vs 초기 기업 60%가 30일 초과).

## 2. Requirements for ML Systems
> **Reliability**: 장애(hardware/software fault, human error)에도 올바른 성능 유지. ML은 ground truth 없이 silent failure가 가능해 "correctness" 판단이 어렵다.
> **Scalability**: complexity(model 크기), traffic volume, model count 측면에서 성장. resource scaling(up/down-scaling, autoscaling)뿐 아니라 artifact management(100개 model 자동 monitor·retrain·재현)도 필요.
> **Maintainability**: ML engineer·DevOps·SME가 각자 편한 tool로 협업할 수 있게 code 문서화, code/data/artifact versioning, reproducibility 확보.
> **Adaptability**: shifting data distribution과 business requirement에 service 중단 없이 update·개선할 수 있는 능력.

## 3. Iterative Process
> ML system 개발은 never-ending cycle로, step 간 back-and-forth가 많다(metric 선택→data 수집/labeling→feature→training→error analysis→relabel/추가수집→배포→business 피드백→objective 변경→처음으로).

```
1. Project scoping
2. Data engineering
3. ML model development
4. Deployment
5. Monitoring & continual learning
6. Business analysis  -> (loops back to 1)
```

## 4. Framing ML Problems
> "느린 고객지원"은 문제이지 ML problem이 아니다. ML problem은 input, output, objective function 3요소로 정의된다.
> 예: 고객 요청을 4개 부서로 routing → classification problem(input=요청, output=부서, objective=예측·실제 부서 차이 최소화).

## 5. Types of ML Tasks
> output이 task type을 결정. 대분류는 classification과 regression이며 서로 변환 가능(house price를 bucket으로 quantize → classification; spam을 0~1 출력+threshold → regression).
> binary < multiclass(class 적을수록 단순). class 수가 매우 많으면 high cardinality 문제(class당 최소 ~100 example 필요, rare class 데이터 수집 난해) → hierarchical classification으로 완화.
> multilabel(한 example이 여러 class): one-hot 다중화([0,1,1,0]) 또는 class별 binary model 집합. label multiplicity(annotator 불일치)와 raw probability에서 prediction 추출이 어려움.

## 6. Multiple ways to frame a problem
> framing 변경이 난이도를 크게 바꾼다. "다음에 열 app 예측"을 N개 app에 대한 multiclass(output=N차원 벡터)로 짜면 새 app마다 retrain 필요.
> regression으로 reframe(input에 app feature 포함, output=0~1 단일 값) 하면 새 app 추가 시 retrain 없이 입력만 추가하면 됨.

## 7. Objective Functions
> 학습을 guide하는 objective(loss) function은 wrong prediction의 loss를 최소화. supervised에선 model 출력과 ground truth를 RMSE/cross entropy 등으로 비교.
> 통상 common loss 사용: regression=RMSE/MAE, binary=logistic(log) loss, multiclass=cross entropy.

```python
def cross_entropy(p, q):
    return -sum([p[i] * np.log(q[i]) for i in range(len(p))])
# p = ground truth, q = predicted distribution
```

## 8. Decoupling objectives
> 여러 objective(spam/NSFW/misinformation filter, quality rank, engagement rank)가 충돌할 때(engaging하지만 저품질 post), 두 가지 접근:
> (1) loss 결합 후 단일 model: `loss = α·quality_loss + β·engagement_loss` → α,β 조정마다 retrain 필요.
> (2) objective별 model 분리(quality_model, engagement_model) 후 출력 결합: `α·quality_score + β·engagement_score` → retrain 없이 tuning 가능. 유지보수 주기도 분리 가능(spam은 quality보다 빠르게 진화).

## 9. Mind Versus Data
> 지난 10년의 진보는 algorithm 개선보다 data 관리/개선에 의존했다. mind 진영(Judea Pearl "Data is profoundly dumb", Christopher Manning: 구조가 적은 data로 더 학습)과 data 진영(Rich Sutton "The Bitter Lesson", Norvig "We just have more data") 논쟁.
> 핵심 쟁점은 finite data가 필요한가가 아니라 충분한가. dataset 크기는 급증(1B Word→GPT-2 10B→GPT-3 500B tokens)했으나, 저품질·outdated·오라벨 data는 오히려 성능을 해친다.

---

## Summary (핵심 정리)
- 모든 ML project는 "왜"에서 시작해 business objective를 ML objective로 번역해야 하며, business metric을 움직이지 못하는 ML metric 개선은 무의미하다.
- 좋은 ML system은 reliability·scalability·maintainability·adaptability를 만족해야 하고, 개발은 선형이 아니라 monitoring·재학습을 포함한 끝없는 iterative cycle이다.
- ML problem은 input/output/objective function으로 framing되며 framing·task type 선택이 난이도를 좌우하고, 여러 objective는 decouple하는 것이 개발·유지보수에 유리하다. 최종적으로 ML 성공은 data의 quality·quantity에 크게 의존한다.
