# 11. The Human Side of Machine Learning

## 챕터 개요 (3줄 요약)
- ML system의 probabilistic·mostly correct·high-latency 특성은 user experience(consistency, 다중 예측, smooth failing)에 영향을 준다.
- ML 팀 구조는 SME와의 cross-functional 협업과 end-to-end data scientist(별도 Ops 팀 vs data scientist 소유)의 trade-off가 핵심이다.
- Responsible AI(fairness, privacy, transparency)는 추상이 아닌 필수 실천이며, bias 원인 파악·trade-off 이해·조기 대응·model card로 구현한다.

---

## 1. User Experience — Consistency
> ML은 deterministic이 아니라 probabilistic(같은 입력도 다른 결과), 어떤 입력에 맞을지 모름, 느릴 수 있음.
> 사용자는 consistency 기대. Booking.com 필터 추천: 매번 다른 필터 추천은 혼란 → 같은 추천 반환 조건(필터 적용 시)과 새 추천 조건(목적지 변경 시) rule. consistency-accuracy trade-off.

## 2. Combatting "Mostly Correct" Predictions
> GPT-3 등 large language model은 task별 학습 없이 광범위 예측(웹페이지 요구→React code) 가능하나 항상 정확하진 않음.
> mostly correct 예측은 사용자가 쉽게 고칠 수 있을 때 유용(고객지원 응답 편집). 고칠 수 없으면(React 모르는 사용자) 무용.
> 해결: 같은 입력에 여러 예측을 보여 적어도 하나는 맞을 확률↑, 비전문가도 평가 가능하게 렌더링(human-in-the-loop AI).

## 3. Smooth Failing
> 빠른 model도 특정 query(긴 sequence)엔 오래 걸림 → backup system(heuristic, 단순 model, cached 예측)으로 "main이 X ms 초과 시 backup 사용" rule. 또는 소요 시간 예측 model로 라우팅(추가 latency).
> speed-accuracy trade-off: 덜 정확하나 빠른 model을 latency 중요 상황에 사용. backup system으로 둘 다 가능.

## 4. Team Structure — Cross-functional Collaboration
> SME(의사·변호사·은행원 등)는 ML system의 사용자이자 개발자. labeling뿐 아니라 problem formulation, feature engineering, error analysis, evaluation, UI 전반에 기여 필요.
> 난제: 비엔지니어 SME에게 ML 한계·역량 설명, domain expertise를 code로 version화("의사에게 Git 사용 기대"). SME를 초기부터 참여시키고 no-code/low-code platform으로 권한 부여(현재 labeling/QA/feedback 중심, 확대 중).

## 5. End-to-End Data Scientists
> MLOps는 ML + Ops(deployment, containerization, orchestration) 전문성 필요. 두 접근:
> **접근 1 — 별도 production 팀**: 채용 쉬움·집중. 단점: 의사소통 overhead, debugging 곤란, finger-pointing, narrow context(전체 가시성 부재).
> **접근 2 — data scientist가 전 과정 소유**: "grumpy unicorn"(모든 것 알기 기대, boilerplate 多). infrastructure는 data science와 다른 skill("app 개발자에게 Linux kernel 기대" 비유).
> 핵심은 좋은 tool/infrastructure(Netflix: specialist가 자기 부분 자동화 tool 생성 → data scientist가 end-to-end 소유). 데이터에 집중하고 싶지 AWS instance·Dockerfile·YAML debugging 아님.

## 6. Responsible AI — 정의 & 사례
> good intention·충분한 인식으로 AI를 설계·개발·배포해 사용자 권한 부여·신뢰·공정·긍정적 영향 보장. fairness, privacy, transparency, accountability.
> **사례 I (UK A-level auto-grader)**: 60% 정확도. 3대 실패 — ① 잘못된 objective(학생 정확도 아닌 학교 간 "기준 유지" 최적화 → 저성과 학교 우수 학생 강등), ② fine-grained evaluation 부족(소규모 학교는 teacher 평가만 → 사립학교 유리, demographic bias), ③ transparency 부재(objective·model을 결과 발표일까지 비공개).
> **사례 II (Strava anonymized data)**: 익명화에도 군 기지 활동 노출. opt-out 기본 설정·불명확한 privacy 설정 문제 → opt-in이 기본이어야. 기기가 끊임없이 data 수집·저장이 근본 위험.

## 7. A Framework for Responsible AI
> **bias 원인 발견**: training data(대표성), labeling(주관적 경험), feature engineering(disparate impact — 보호 계급과 상관된 변수; DisparateImpactRemover, Infogram), model objective(다수 편향), evaluation(slice-based).
> **data-driven 한계 이해**: data만으론 부족, 사회경제·문화 고려, domain expert와 협업(학제 경계 넘기).
> **desiderata trade-off**: privacy vs accuracy(differential privacy는 underrepresented class 정확도 더 하락), compactness vs fairness(compression이 long-tail 보호 feature에 disparate impact; pruning이 quantization보다 큼).

```
differential privacy: higher privacy -> lower accuracy (worse for minorities)
compression:          pruning > quantization in disparate impact
```

## 8. Responsible AI 실천
> **act early**: bias를 일찍 고려할수록 비용↓(NASA: 오류 비용은 lifecycle 단계마다 10배 증가).
> **model card**: 학습·평가 방법, 의도된 사용 맥락·한계 문서화(model details, intended use, factors, metrics, evaluation/training data, ethical considerations). model 업데이트마다 갱신 필요 → 자동 생성 tool(TF/Metaflow/sklearn). model store가 자동 생성으로 진화 가능.
> **bias 완화 process 수립**: ad hoc일수록 오류↑. 내부 tool portfolio(Google best practice, IBM AIF360), 제3자 audit.
> **최신 동향 유지**: ACM FAccT, Partnership on AI, Alan Turing Institute, AI Now Institute.

---

## Summary (핵심 정리)
- ML의 probabilistic·mostly correct·high-latency 특성은 user experience에 영향을 주며, consistency rule·다중 예측 제시·backup system(smooth failing)으로 대응한다.
- ML 팀은 SME와의 cross-functional 협업이 필요하고, end-to-end data scientist 접근(별도 Ops 팀의 overhead vs data scientist 소유의 skill 부담)은 좋은 tool/infrastructure로 가능해진다.
- Responsible AI는 필수 실천이며 bias 원인(data/labeling/feature/objective/evaluation) 파악, trade-off(privacy-accuracy, compactness-fairness) 이해, 조기 대응, model card, 완화 process 수립으로 구현한다.
