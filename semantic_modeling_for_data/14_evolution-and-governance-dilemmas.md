# 14. Evolution and Governance Dilemmas

## 챕터 개요 (3줄 요약)
- semantic model은 품질·유용성 유지를 위해 진화해야 하는 dynamic artifact이며, evolution 전략과 governance system이 모두 필요하다.
- evolution dilemma: remember/forget, run/pace, react/prevent, 그리고 semantic drift 측정·대응.
- governance dilemma는 centralized와 decentralized 사이의 정도를 여러 factor로 결정하는 것이다.

---

## 1. Model Evolution
> first version이 모든 dimension에서 충분하고 외부 force도 없을 때만 evolution을 무시할 수 있다.

### 1.1 Remember or Forget?
> 제거한 statement를 삭제할지 "재활용"해 보존할지의 dilemma다.

- 보존이 유용한 경우: ① 재출현 위험 높음(root cause 못 고치면 negative 버전으로 차단 — Ch13), ② historical 지식 가치(옛 수도를 former capital로, 2019년 직원 수를 time-contextualized로).

### 1.2 Run or Pace?
> 잦은 작은 release vs 드문 큰 release의 dilemma다.

- EMSI는 2주마다, FIBO는 분기마다. pace 결정 factor: 변경 필요 속도, 구현 가능 속도, 변경 가치/overhead 비율, **의존 artifact의 흡수 속도**(reindex 3주면 주간 release 곤란).
- 방법: release-worthiness 기준 정의(특정 정확도 fix는 즉시, 20%+ statement 추가 시), 의존 artifact의 change tolerance 파악(가장 안 받는 것이 최대 빈도 결정), **skip-safe**하게(중간 버전 건너뛰어도 안전 — Textkernel은 분기 client용으로 biweekly release를 호환되게).
- **알 수 없는 의존자 존중**: 공개 model(DBpedia·SNOMED)은 모르는 의존자 多 — 투명·문서화된 evolution 전략 공유가 좋은 관행.

### 1.3 React or Prevent?
> update가 사용자 feedback 결과(reactive)인지 미리 추측(preventive)인지의 dilemma다.

- **reactive**는 쉽고 저렴하나 feedback 메커니즘이 comprehensive·reliable·timely해야 — feedback이 model에 관한 건지 app에 관한 건지 분리, model semantics와 호환, 쉽고 가치 있게(이메일 폼만으론 안 됨; Amazon 추천 feedback이 불편해져 중단한 일화).
- **preventive**는 도구·process로 적시에 update 필요 감지 — 문서당 평균 탐지 entity 하락(completeness↓), 특정 언어 lexicalization 없는 entity 비율(언어 divergence), outlier 증가(accuracy↓), 잘못된 link 증가(understandability↓), 다운로드 하락(가치↓).
- 균형 4행동: 각 부분의 품질 저하 susceptibility 평가(resilient하면 reactive 가능), risk·benefit으로 중요도 평가(critical하면 proactive), feedback 메커니즘 효과 평가(약하면 proactive), 받은 feedback으로 근본 원인 수정·자동 탐지 개선.

### 1.4 Knowing and Acting on Your Semantic Drift
> 가장 volatile해 잦은 update가 필요한 부분을 식별하려면 semantic drift를 정의·측정해야 한다.

- 2가지 인식: ① drift 정의는 model의 content·domain·context를 반영해 적응해야(generic formalization이 다 적용되진 않음), ② 단일 최적 측정법이 아닌 여러 방법이 다른 해석·용도를 가짐.
- **Drift modeling**(노동시장 KG 예): profession·skill·qualification 등 entity, 다수 vague relation(truth degree·applicability context·provenance attribute). journalist처럼 기술 진화로 의미 변화. drift는 lexicalization·**intension**·extension으로 모델링하나, skill·profession은 abstract라 extension은 부적합 — intension이 핵심. 모든 attribute/relation이 동등 기여하지 않음(preferred label > alt label, broader > narrower, profession은 essential skill이 핵심).
- **Drift measuring**: 시점 간 의미 유사도 차이로 정량화. lexicalization은 string이 아닌 set similarity(철자 변화는 drift 아님), vague relation은 fuzzy degree 고려(top N 관련 concept을 Kendall Rank로 — Data Scientist의 top 10 skill 순위 변화 감지). **versatile framework** 필요(target type·time scope·relation·context·provenance 파라미터) — 같은 concept도 CV(노동 공급)·vacancy(산업)·news(일반 인식)·Wikipedia(핵심 의미)별로 다른 drift. 관계·context별 importance weight로 aggregate.
- **사용자 관점**: drift는 model 사용자에게도 유용 — 노동시장 변화를 구직자·교육기관·정책입안자에게 전달(같은 직함이라도 내용이 달라진 직업 인식 갱신).

---

## 2. Model Governance
> evolution 전략 실행을 보장하는 원칙·process·규칙 체계로, 근본 dilemma는 centralized vs decentralized 정도다.

### 2.1 Democracy, Oligarchy, or Dictatorship?
> centralized(소수 동질 팀)와 decentralized(다수 이질 그룹, 중앙 권위 없음) 사이를 factor로 결정한다.

- 양측 논거(전문성 vs 협업 확장성·다양성 vs 목표 조준)는 모든 상황에 적용되지 않음. 진단 factor: 개발 자동화 정도(높으면 적은 기여자), model modularity(독립 컴포넌트 많으면 decentralize 가능), 품질 관리 자동화 정도, 요구되는 breadth·diversity·vagueness(넓으면 전문가 소수로 어려움), 품질 우선순위(completeness 우선이면 느슨하게), 변경의 downstream 영향(크면 strict), 공유 goal·원칙 채택, model 복잡성, 갈등 해결·결정 메커니즘 효율, 조직의 기존 data governance 문화.
- 이 factor들로 governance task·aspect별 다른 (de)centralization 정도 정의. **clean slate vs legacy**: 신규는 central에서 시작해 점진 decentralize, 기존 model 통합 시 각자 governance 무시 말 것.

### 2.2 A Centralization Story (Textkernel)
> 분산되고 비공식적이던 여러 model의 governance를 통합하며 centralize.

- 문제: 4개 팀이 공유 관행·지침 없이 편집 — skill·synonym 해석 차이(accuracy 오류), 중복 추가(conciseness 훼손), 무분별한 custom 버전(유지 overhead).
- 변경: 책임 팀 5→2개로 축소, 부분·변경 유형별 다른 process·규칙(schema 변경은 더 신중), 핵심(특히 vague) element 정의 조화·중앙 문서화·이름 명확화, 자동 품질 control·alert 증가(더 많은 기여 허용), custom 기준·process 엄격화.

---

## Summary (핵심 정리)
- 유지·진화를 효과적으로 계획·우선순위화하려면 semantic drift의 성격·강도를 이해하고 대응해야 한다.
- drift를 모델링·측정할 땐 model의 content·domain·application context를 반영해 적응한다.
- centralized governance가 본질적으로 더 좋거나 나쁘지 않으며, modularity·개발 자동화 정도 등으로 task·aspect별 (de)centralization 정도를 정한다.
