# 09. Continual Learning and Test in Production

## 챕터 개요 (3줄 요약)
- continual learning은 retraining 빈도가 아니라 방식(stateless retraining vs stateful training)에 관한 것이며 대체로 infrastructure 문제다.
- data freshness의 가치를 실험으로 측정해 retraining 빈도를 정하고, model iteration과 data iteration을 구분한다.
- offline evaluation만으론 부족하므로 test in production(shadow, A/B, canary, interleaving, bandit)으로 업데이트를 안전히 검증한다.

---

## 1. Continual Learning 개념
> 매 sample마다 update는 거의 안 함(neural network는 catastrophic forgetting, batch hardware 비효율). 실제론 micro-batch(512/1024)로 update.
> champion(기존) vs challenger(업데이트된 replica): replica를 새 data로 학습해 더 나을 때만 교체. 빈도가 핵심이 아님.

## 2. Stateless Retraining vs Stateful Training
> **stateless retraining**: 매번 from scratch(대부분 기업). **stateful training**(fine-tuning, incremental): 마지막 checkpoint에서 새 data로 계속 학습.
> stateful은 적은 data로 update 가능(어제 checkpoint + 하루치). Grubhub: stateful로 compute 45배↓·purchase-through 20%↑. data 저장 없이도 가능(privacy↑). 가끔 from scratch로 calibration.
> **model iteration**(feature/architecture 변경) vs **data iteration**(동일 구조, 새 data refresh). stateful은 주로 data iteration(model iteration은 knowledge transfer/model surgery 연구 단계).

```
stateless: train from scratch each time (more data)
stateful:  continue from checkpoint (less data, cheaper)
```

## 3. Why Continual Learning?
> ① 갑작스런 data distribution shift 대응(Lyft 동네 이벤트로 수요 급증), ② rare event 적응(Black Friday, Alibaba Singles Day), ③ continuous cold start(신규 사용자뿐 아니라 device 전환·미로그인·드문 방문; 70%+ shopper가 연 3회 미만 방문). TikTok은 수분 내 사용자 적응. continual learning은 batch learning의 superset.

## 4. Challenges
> **fresh data access**: data warehouse는 느림 → real-time transport(Kafka/Kinesis)에서 직접 pull. label도 필요(natural label·short feedback loop task가 최적: dynamic pricing, ETA, ad CTR, recommender). label computation(log 되짚어 추출, 비용 큼 → stream processing). Snorkel·crowdsourcing으로 가속.
> **evaluation**: 가장 큰 난제는 update가 배포할 만큼 좋은지 보장. 빈번할수록 실패 기회↑·adversarial 취약(Microsoft Tay 16시간 만에 중단). 평가 시간이 빈도 bottleneck(fraud는 imbalance로 2주 소요).
> **algorithm**: matrix-based(collaborative filtering)·tree-based는 빠른 update 어려움(전체 dataset 필요). neural network는 어떤 batch size든 update 가능. Hoeffding Tree 등 존재. feature scaling 통계도 online으로 incremental 계산 필요.

## 5. Four Stages of Continual Learning
> **stage 1 — manual stateless retraining**: 새 model 개발 우선, update는 수동·ad hoc(분기/반기). 대부분 비테크 기업.
> **stage 2 — automated retraining**: script로 retraining 자동화(Spark batch), 빈도는 직감. 필요: scheduler(Airflow/Argo), data 접근성, model store. model 간 다른 빈도·의존성 고려. (feature reuse "log and wait"로 train-serving skew 완화.)
> **stage 3 — automated stateful training**: checkpoint 로드 후 계속 학습. 필요: mindset 변화 + data/model lineage 추적(in-house).
> **stage 4 — continual learning**: 고정 schedule 대신 trigger(time/performance/volume/drift-based)로 자동 update. edge deployment와 결합이 holy grail. 견고한 monitoring + 평가 pipeline 필요.

## 6. How Often to Update Your Models
> **value of data freshness**: 과거 다른 time window로 학습해 오늘 data로 평가(model A: Jan-Jun, B: Apr-Sep, C: Jun-Nov → Dec 테스트)해 fresher data의 gain 측정. Facebook: weekly→daily로 ad CTR loss 1%↓(충분히 유의). 일부 기업은 수분마다.
> **model vs data iteration**: data iteration이 gain 적으면 더 나은 model에, model iteration이 100X compute로 1%면 data iteration(1X)이 유리. 실험으로 결정. 초기엔 "가능한 자주".

## 7. Test in Production — 한계
> **test split**(static benchmark): 새 분포 적응 model엔 불충분. **backtest**(최근 기간 data로 평가): pipeline 오류 시 부족 → static test set으로 sanity check 병행. 배포해야만 production 성능 확인 가능 → test in production 필요.

## 8. Test in Production 기법
> **shadow deployment**: candidate를 병렬 배포, 모든 요청을 양 model에 라우팅하되 기존 model 예측만 제공·새 model log. 가장 안전하나 비용 2배.
> **A/B testing**: traffic을 무작위 분할(selection bias 금지)해 양 model 비교, 충분한 sample로 통계적 유의성(two-sample test). p-value 해석 주의. A/B/C도 가능.
> **canary release**: candidate(canary)에 점진적 traffic 증가, 성능 나쁘면 중단·롤백. A/B testing 구현에 사용 가능하나 무작위 불필요.
> **interleaving**: 한 사용자에게 양 model 추천을 섞어 노출해 선호 측정(Netflix: A/B보다 적은 sample). 위치 영향 제거 위해 team-draft interleaving.
> **bandits**: model을 slot machine으로 보고 exploitation/exploration 균형. stateful(현재 성능 계산 필요), online prediction·short feedback loop 필요. A/B보다 data 효율적(630K vs 12K sample). ε-greedy, Thompson Sampling, UCB. 구현 어려워 빅테크 위주.
> **contextual bandits**: 각 action(추천 item)의 payout 결정, partial/bandit feedback 문제 해결(보여준 item만 feedback). "one-shot" RL. data 효율↑이나 model 구조 의존적이라 구현 더 어려움.

> 평가는 "무슨 test"뿐 아니라 "누가 test하는가"도 중요 — data scientist ad hoc 평가는 bias·가변성 → 자동화된 명확한 pipeline(CI/CD 유사) 필요.

---

## Summary (핵심 정리)
- continual learning은 retraining 빈도가 아니라 stateless retraining vs stateful training의 방식 문제이며, 대체로 streaming infrastructure를 요하는 infrastructure 과제다.
- 4단계(manual → automated retraining → stateful → continual)로 성숙하며, data freshness의 가치를 실험으로 측정해 retraining 빈도와 model/data iteration을 결정한다.
- offline evaluation(test split/backtest)만으론 부족하므로 shadow·A/B·canary·interleaving·bandit으로 production에서 안전하게 검증하고, 평가 pipeline을 자동화한다.
