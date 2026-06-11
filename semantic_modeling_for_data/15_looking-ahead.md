# 15. Looking Ahead

## 챕터 개요 (3줄 요약)
- 본서는 "완벽한 model" 레시피가 아닌 pitfall·dilemma 구조를 택했는데, 저자의 map이 독자의 territory를 반영하지 않기 때문이다.
- semantics는 consensus 기반이라 어렵다는 현실을 직시하되, tunnel vision·소모적 논쟁을 피하고 두 paradigm의 장점을 결합해야 한다.
- bias로 누구도 harm하지 않도록 하고, data 공급·활용 측의 semantic gap을 좁히는 것이 최종 사명이다.

---

## 1. The Map Is Not the Territory
> 완벽한 model 레시피를 주지 않은 이유는 독자의 도메인·data·context를 모르기 때문이다.

- 함께 맞춤 전략(Ch11)을 짜지 않는 한, 특정 언어·dimension을 권하는 건 무책임. 대신 context의 위험과 회피·완화법을 알려 독자가 스스로 길을 내고 navigate하게 함.

---

## 2. Being an Optimist, but Not Naïve
> semantic modeling은 기술 발전에도 아직 해결되지 않은 어려운 문제다.

- 비관이 아닌 현실 — semantics는 거의 항상 **consensus 기반**이고 consensus 구축은 어렵다. model은 그 consensus 범위만큼만 가치 있고, 동의 안 하는 측의 올바른 사용은 보장 못 함. 규모·다양성이 커질수록 consensus 표현 능력이 압박받음.
- Semantic Web 비판(Clay Shirky 2003, Ghent 연구자 2019 "Semantic Web Identity Crisis")이 이를 지적. 기술 가치가 없다는 게 아니라, "하룻밤에 모든 data를 상호운용 가능하게 만든다"는 vendor 주장에 면역돼야 한다는 것.

---

## 3. Avoiding Tunnel Vision
> 연구처럼 task를 고립해 풀면 일차원적·불완전하며, 전체로 결합해야 이긴다.

- 가장 표현력 있는 언어도 modeler가 잘못 쓰면 무용, billions statement 저장·reasoning infra도 정확히 획득 못 하면 무용, 최고의 model도 evolution 전략 없으면 decay. 특화 방법·도구는 중요하나 전체로 결합돼야 함.

---

## 4. Avoiding Distracting Debates
> "내 framework이 더 semantic하다"·"내 ML이 더 진짜 AI다" 같은 논쟁은 무의미·비생산적이다.

### 4.1 Semantic vs Nonsemantic Frameworks
> framework을 semantic/비semantic으로 나누려면 Semantic Modeling Framework class를 비vague하게 정의해야 하나, 그 필요충분조건을 못 찾으면 단정할 수 없다.

- semantic distinction은 가치 있을 때만 — framework이 비semantic임을 증명하는 건 무가치, 필요한 model을 짓는 데 얼마나 도움 되는지가 가치(CSV로 충분하면 왜 걱정?).
- 각 framework의 장단: RDF(S)/OWL은 URI로 web 식별·interlink 쉬우나 OWA로 고립 통제 어려움. Neo4j(property graph)는 class/individual 구분 없으나 relation attribute 쉬움. 표준 element(rdfs:subClass) 지원이 "더 semantic"의 근거일 수 있으나, 모두가 올바로 해석·사용할 때만 가치(아니면 ad hoc element도 똑같이 좋음).
- framework이 semantic인지 집착 말고 어느 게 최선인지·올바른 사용법을 배우는 게 생산적.

### 4.2 Symbolic Knowledge Representation vs Machine Learning
> symbolic model은 명시적 symbol·logic(human 먼저), ML은 subsymbolic latent(machine 먼저)로 의미를 표현한다.

- ML 측은 symbolic이 generalize·scale 못 한다 비판, symbolic 측은 ML이 미묘한 distinction·설명 불가라 비판. 답은 둘 다 아님 — app에 따라 적합성 다름(영화 추천은 ML, 의료 진단·법률은 symbolic의 distinction·설명).
- **complement**: 개발·진화 자동화엔 ML 기반 mining 필요, semantics·deductive reasoning은 ML data를 pre-label·설명·training 밖 상황 대응. 향후 ML이 symbolic 필요를 없앨 수도 있으나 당분간 둘의 장점을 취함.

---

## 5. Doing No Harm
> semantic model은 인간 신념의 추상화라 human bias에 취약하며, crisp fact라고 ML보다 덜 biased인 건 아니다.

- 추상·vague·context-dependent 지식(Data Scientist vs Engineer, Old/Young 나이 기준)이 의도치 않은 결과 초래(신뢰 source의 허위 정보, 소외 집단 삭제 — Old Person 정의가 보험료에 쓰이면?).
- bias는 specification, 구축 algorithm·source, 품질 측정·우선순위, evolution 전략 어디서든 발생하고 사용 규모에 따라 증폭. 모든 결정마다 누가 harm받는지 생각·테스트, method·source·가정·design을 scrutinize·투명 문서화. 사용자는 model을 액면 그대로 받지 말고 bias를 능동 식별. bias의 악순환을 선순환으로(bias-aware model이 ML을 덜 biased하게, 역도).

---

## 6. Bridging the Semantic Gap
> data 공급·활용 측의 semantic gap을 좁히는 것이 양측 이익이나 매우 어려운 사명이다.

- 한 이유는 human language의 ambiguity·vagueness·context dependence·drift — 새 방법 연구 외엔 어쩔 수 없음.
- 다른 이유는 이미 가진 방법을 **suboptimal하게 적용**하는 것 — 본서가 기술을 더 잘 써서 gap을 좁혀 synergy·reuse·consistency의 결실을 맺는 법을 보였다. "공은 이제 독자의 코트에 있다."

---

## Summary (핵심 정리)
- "map은 territory가 아니다" — 완벽한 model은 도메인·context마다 다르므로 pitfall·dilemma 회피·완화법을 익혀 스스로 길을 낸다.
- semantics는 consensus 기반이라 어렵다는 현실을 직시하되, 과장된 vendor 주장에 면역된다.
- task 고립(tunnel vision)과 framework·paradigm 우열 논쟁을 피하고, symbolic과 ML의 장점을 결합한다.
- bias로 누구도 harm하지 않도록 결정마다 점검·문서화하며, 궁극적으로 semantic gap을 좁힌다.
