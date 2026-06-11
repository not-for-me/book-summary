# 01. Overview of Machine Learning Systems

## 챕터 개요 (3줄 요약)
- ML system은 algorithm만이 아니라 business requirement, interface, data stack, 개발/monitoring/update logic, infrastructure를 모두 포함하는 holistic system이다.
- ML이 적합한 문제는 "complex pattern을 existing data에서 학습해 unseen data에 prediction"하는 경우이며, 반복적·대규모·저비용 오류·변화하는 pattern일수록 효과적이다.
- production ML은 research ML 및 traditional software와 본질적으로 다르며(stakeholder, latency, data, fairness, interpretability, code+data 결합), 이 차이가 이 책의 system 접근법의 출발점이다.

---

## 1. ML systems design과 MLOps의 관계
> MLOps는 ML을 production에 올리는(deploy·monitor·maintain) tool/best practice 집합이고, ML systems design은 모든 component와 stakeholder가 objective를 만족하도록 system 전체를 보는 접근법이다.

## 2. When to Use Machine Learning
> ML = (1) learn (2) complex pattern을 (3) existing data로부터 학습해 (4) prediction을 (5) unseen data에 수행하는 접근. 5개 키워드 각각이 적용 가능 조건을 규정한다.
> - learn: 학습할 capacity가 있어야 함(relational DB는 ML 아님).
> - complex pattern: 단순 lookup(zip→state)은 ML 불필요, 복잡한 관계(rental price)에 적합. ML = "Software 2.0".
> - existing data: 학습할 data가 있거나 수집 가능해야 함. zero-shot도 다른 task data로 사전학습 필요.
> - prediction: predictive answer가 필요한 문제만. compute-intensive 문제를 approximate prediction으로 reframe 가능.
> - unseen data: train/unseen data가 유사 distribution을 공유해야 의미 있음.

## 3. ML이 특히 빛나는 추가 조건
> repetitive(반복 pattern은 학습 쉬움), 오류 비용이 cheap(recommender처럼), at scale(대량 prediction으로 초기 투자 정당화), 그리고 pattern이 계속 변함(hardcoded rule은 금방 outdated, ML은 new data로 갱신).

## 4. ML을 쓰면 안 되는 경우
> 비윤리적이거나, 더 단순한 해법으로 충분하거나(Ch6: ML model 개발 첫 단계는 non-ML solution), cost-effective하지 않을 때. 단, 전체가 안 되어도 문제를 쪼개 일부 component에 ML 적용 가능.

## 5. Machine Learning Use Cases
> consumer(search, recommender, 예측 typing, 얼굴/지문 인증, 번역, smart assistant)보다 enterprise use case가 다수. enterprise는 보통 정확도 요구가 엄격하나 latency엔 관대.
> 대표 enterprise: fraud detection, price optimization, demand forecasting, customer acquisition 비용 절감, churn prediction, support ticket classification, brand monitoring(sentiment analysis), health care 진단.

## 6. ML in Research Versus in Production
> 5대 차이: requirement(SOTA benchmark vs 다양한 stakeholder의 상충 요구), computational priority(fast training/throughput vs fast inference/low latency), data(static vs 계속 shift), fairness(비초점 vs 필수), interpretability(비초점 vs 필수).
> restaurant recommender 예: ML engineer·sales·product·platform·manager가 각기 다른 objective → 여러 objective를 만족시키려면 objective decoupling(objective별 model 후 prediction 결합).
> ensembling은 competition에선 강하나 production에선 복잡도 때문에 잘 안 쓰임.

## 7. Computational priorities & Latency
> production은 latency가 핵심(Akamai: 100ms 지연→conversion 7%↓; Booking.com: latency 30%↑→conversion 0.5%↓).
> latency는 단일 값이 아니라 distribution → average 대신 percentile(p50/p90/p95/p99)로 봐야 outlier·고가치 사용자(데이터 많은 Amazon 고객) 파악 가능.
> batching: query 1개씩 처리 시 latency↑=throughput↓이지만, batch 처리 시 latency↑가 throughput↑일 수 있음(trade-off).

```
one-at-a-time:  higher latency -> lower throughput
batched:        higher latency -> possibly higher throughput
```

## 8. ML Systems Versus Traditional Software
> SWE는 code와 data를 분리(separation of concerns)하지만, ML system은 part code + part data + part artifact. data가 빠르게 변하므로 적응적 개발/배포 cycle 필요.
> code뿐 아니라 data도 test·version해야 함(어려운 부분). data sample은 동등하지 않음(희귀 cancerous lung scan이 normal보다 가치 큼; data poisoning 위험).
> 대형 model(수억~수십억 param)을 edge device에 올리고 충분히 빠르게 돌리는 것, production monitoring/debugging이 ML 고유 난제.

---

## Summary (핵심 정리)
- ML system은 algorithm을 넘어 data stack·deployment·monitoring·infra까지 포함하는 전체 system이며, 이 책은 그 holistic system 접근법(framework)을 제공한다.
- ML 적합성은 "complex pattern을 data로 학습해 unseen data에 prediction"이라는 정의로 판별하며, 반복적·대규모·저비용 오류·변화하는 pattern일수록 효과적이다.
- production ML은 research와 달리 stakeholder 충돌, low latency, shifting data, fairness, interpretability를 모두 고려해야 하고, traditional software와 달리 code+data+artifact가 결합되어 data versioning/testing이 핵심 난제다.
