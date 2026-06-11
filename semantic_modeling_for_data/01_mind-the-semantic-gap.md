# 01. Mind the Semantic Gap

## 챕터 개요 (3줄 요약)
- 데이터 공급(supply) 측과 활용(exploitation) 측 사이의 **semantic gap**(의미 이해의 불일치)이 data 가치 창출을 가로막는 핵심 문제다.
- semantic data modeling은 data의 meaning을 humans와 machines 모두에게 명시적·정확·공통적으로 이해되도록 표현하는 작업이다.
- 좋은 semantic model은 적절한 expressiveness/clarity의 균형을 찾는 것이며, 그 과정은 pitfall(명백한 실수)과 dilemma(trade-off 선택)로 위협받는다.

---

## 1. What Is Semantic Data Modeling?
> semantic modeling은 data의 의미를 humans와 machines가 공통으로 이해하도록 명시적·정확하게 표현하는 것이다.

- semantics는 signifier(words, signs, symbols)와 그것이 denote하는 것(entity, concept, idea) 사이 관계를 다루는 의미 연구. 공통 이해를 만들어 서로를 이해하게 함.
- semantic model에 포함되는 artifact: metadata schema, controlled vocabulary, taxonomy, ontology, knowledge graph, E-R model, property graph 등 conceptual model.
- 예: **SNOMED CT**(의료 용어 concept·synonym·definition·hierarchical relation), **ESCO**(EU 노동시장의 occupation·skill·qualification 다국어 분류).
- knowledge graph·ontology의 정확한 정의·차이엔 논쟁이 있으나, 본서는 "의미를 명시적·공통 이해되게 하는 모든 data 표현"을 semantic model로 통칭.
- **machine learning model은 제외**: ML은 의미 명시가 목적이 아닌 subsymbolic latent 표현(crisp하지 않은 통계적 규칙성). semantic model은 symbolic·discrete fact·정확한 identity. 둘은 우열이 아닌 **complementary** — ML이 semantic model 개발을 자동화하고, semantic model이 ML을 강화할 수 있음.

---

## 2. Why Develop and Use a Semantic Data Model?
> AI/data science 기능 강화와 silo data의 의미 표준화·정렬·discoverability 향상을 위해 semantic model을 사용한다.

- ontology·knowledge graph는 수십 년 됐으나 최근 급부상(Google 2012 "things, not strings", Gartner 2018 hype cycle). Amazon, LinkedIn, Thomson Reuters, BBC, IBM 등이 활용.
- **AI 기능 강화**: ML 기반이어도 explicit symbolic knowledge가 필요한 task 존재. 예: IBM **Watson**(Jeopardy! 우승)은 temporal/geospatial 관계 호환성 판단, 상호배타적 type(사람≠국가) 식별에 ontology 활용.
- **이질적 silo data 표준화·정렬**: 의미 정렬·context 부여로 discoverable·interoperable·usable하게. 예: Thomson Reuters knowledge graph(2017)가 20,000+ source의 organization·people·financial instrument 등을 통합해 data discovery·analytics 지원.
- 핵심: model이 해당 application scenario에서 중요한 meaning 측면을 효과적으로 전달해야 함. 아니면 미사용되거나 잘못 사용됨. 직접 만들지 않은 model을 쓸 때도 그 semantics가 시나리오에 맞는지 확인 필수.

---

## 3. Bad Semantic Modeling
> semantic model은 vague한 지식을 objective fact로 제시하는 등 문제가 될 수 있다.

- ESCO 사례: occupation과 skill 사이 `essential_for`/`optional_for` 관계 제공. 유용하지만 **subjective knowledge를 objective로 제시**하고 vagueness를 사용자에게 알리지 않는 pitfall에 빠짐.
- 문제: 어떤 skill이 직업에 "essential"한지는 대개 **vague**(명확한 적용 기준 부재). 100명에게 물으면 100가지 답. 그런데 ESCO는 모든 context에서 valid한 객관적 사실처럼 제시 → career advice software가 "knowledge engineer가 되려면 web programming 필수"를 불변의 사실로 전달하는 위험.
- semantic modeling이 어려운 이유: human language·perception에 ambiguity, vagueness, imprecision이 가득.
- **핵심 challenge**: 과도한 개발·유지 비용 없이 사용자·application에 유익한 적절한 expressiveness·clarity 수준을 찾는 것. developer/engineer는 meaning을 under-specify, ontologist/linguist는 over-specify하는 경향 → modeler는 균형을 맞춰야 하며, 이는 pitfall과 dilemma에 위협받음.

---

## 4. Avoiding Pitfalls
> pitfall은 data semantics·요구사항·개발 과정에서 명백히 잘못된 결정/행동(또는 필요한 행동의 누락)으로 바람직하지 않은 결과를 낳는 상황이다.

- pitfall은 modeler의 무능·미숙 탓만이 아님. 학계·산업 community도 기여: ① 모순되거나 틀린 terminology 사용·교육, ② 일부 pitfall을 무시·경시, ③ community 자신도 pitfall에 빠진 기술·문헌·model 생산.
- 예: Protégé tutorial은 "concept = class(set of individuals)"라 하고, SKOS spec은 "concept = unit of thought(idea/notion)"라 함 → 모순. 본서는 SKOS 정의가 더 정확하며, OWL의 "concept=class" 주장은 오해를 부르고 여러 modeling error를 유발한다고 봄.
- 본서 목표는 비난이 아니라, model 생성자·사용자 모두가 pitfall을 인식·회피하도록 돕는 것.

---

## 5. Breaking Dilemmas
> dilemma는 각기 장단이 있고 명확한 결정 기준이 없는 선택지 사이에서 골라야 하는 상황이다.

- 예: ESCO의 vague한 `essential_for`를 어떻게 다룰까 — ① "vague"로 flag(기대치는 알려주나 disagreement는 안 줄음), ② context(국가·산업·user group)별 버전 생성(비용·난이도 높으나 불일치 감소).
- 본서는 dilemma에 대한 확정적 "전문가" 해법을 주지 않음 — 그런 것이 없기 때문. 대신 각 dilemma를 **decision-making problem**으로 framing하고, 대안을 feasibility·cost-benefit·strategic 관점에서 평가할 정보를 어디서 찾을지 보여줌.

---

## Summary (핵심 정리)
- semantic gap은 data 공급 측과 활용 측의 의미 이해 불일치로, data 가치 실현을 저해한다.
- semantic data modeling은 data의 meaning을 humans·machines가 공통 이해하도록 명시적으로 표현하며, ontology·taxonomy·knowledge graph 등을 포함한다(ML model은 제외, 둘은 complementary).
- semantic model은 AI 기능 강화와 silo data 표준화·discoverability에 쓰이나, vagueness를 objective로 제시하는 등 bad modeling 위험이 있다.
- 핵심은 적절한 expressiveness/clarity 균형이며, pitfall(명백한 실수 회피)과 dilemma(trade-off를 decision-making으로 framing)가 본서의 두 축이다.
