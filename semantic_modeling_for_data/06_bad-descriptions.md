# 06. Bad Descriptions

## 챕터 개요 (3줄 요약)
- modeler는 machine-interpretability(axiom·inference)에 치중하고 human-interpretability(name·definition·문서)를 과소평가해, 이것이 semantic gap의 가장 큰 원인이 된다.
- 흔한 실수: bad name(ambiguous·inaccurate·vague), definition 누락·부실, vagueness 무시, bias·가정 미문서화.
- vagueness는 flaw가 아닌 feature이므로 제거가 아니라 탐지·명시·문서화해야 한다.

---

## 1. Giving Bad Names
> element 이름은 human이 의미를 이해하도록 돕지 못하거나 잘못 해석하게 하면 bad name이다.

- 예제 dilemma: customer가 person·organization일 때 두 모델 모두 틀림 — 오른쪽은 "모든 customer가 person이자 organization", 왼쪽은 "모든 person·organization이 customer". 진짜 의도는 PrivateCustomer/CorporateCustomer(physical person/organization인 customer).
- **Bad example**: 전문가 모델에도 흔함 — SKOS `skos:broader`(방향 모호, "has broader concept"로 읽어야), DBpedia Agent의 `cost`(너무 generic → hasConstructionCost), Schema.org의 sportsEvent/sportsTeam(parent relation 미반영 → isLocatedAtSportsEvent).
- **왜 bad name을 주는가**: ① ambiguity(다른 해석 모름, 도메인 무관 가정 — Python, context로 추론 기대 — Barcelona-Bernabeu), ② inaccuracy(자연어처럼 부정확하게 표현 — "Car" 대신 "CarModel", "Democracy" 대신 "Democratic Country").
- **Pushing for clarity**: Textkernel에서 search expansion 관계를 모두가 "synonym"이라 부르고 관계명도 belongsTo(무의미) → expandsInSearchQuery로 변경하고 인식도 교정. 기법: 이름을 고립해서 모든 해석 검토(여러 사람·corpus·Google), 다의면 더 specific하게(label은 ambiguous 가능, name은 unambiguous), 정의와 비교, 실제 사용 관찰(오류 많은 relation은 이름 탓), edge case 질문, relation엔 verb phrase·방향 명시(hasBroader).

---

## 2. Omitting Definitions or Giving Bad Ones
> 이름만으론 의미를 알 수 없을 때 textual definition이 필요하다.

- "bug" 비유: 친구가 질병으로 오해(ambiguity → 더 나은 name "software bug"), 의미를 전혀 모름(domain knowledge 부재 → good definition 필요).
- **When you need**: 모든 stakeholder가 같은 배경 지식을 갖는다고 기대할 수 없을 때 — 고도 전문 용어(Agonal), acronym(AWOL), 도메인 특수 의미(financial "ability to pay"). domain qualifier만으론 부족.
- **왜 누락**: "everybody knows" 가정, 난이도·비용(많거나 추상·vague), optionality, "inference에 기여 안 함" 태도(reasoning 전용·human이 쉽게 이해할 때만 유효).
- **Good vs bad definitions**: 부정확(IPTC Fishing Industry="raising or gathering of fish" — industry 누락), circularity(Poetry="forms of poetic expression" — poetry→poetic). class를 "nonclass"로 정의(Democracy 정의에 instance가 country임을 숨김 → "a country governed by...").
- **정의 획득 4방법**: ① 전문가 고용(ESCO, 비싸고 non-scalable), ② 기존 dictionary/glossary 재사용(copyright·품질·도메인 특수성 주의), ③ 텍스트에서 추출(lexico-syntactic pattern, 구조적 텍스트에 유효), ④ 자체 model에서 합성(subclassOf·specializesIn을 verbalize → "A Java developer is a kind of software developer who specializes in Java" — extensional/ostensive/intensional 생성, 어느 instance·relation이 의미에 더 기여하는지가 challenge).

---

## 3. Ignoring Vagueness
> vagueness는 class·relation의 extension을 정확히 결정하기 어렵게 하며, 정의 없이 vague element를 두면 문제가 생긴다.

- 예: DBpedia hasFilmGenre(genre 적용 기준 불명확), Cyc의 Famous Person·Big Building, Business Role Ontology의 Competitor — 추가 정의 없음.
- vague element는 개발·유지·사용자 간 disagreement 유발(electricity market ontology의 Critical System Process — 전문가마다 criticality 기준 다름; Strategic Client — 신임 R&D director의 기준 불일치로 잘못된 결정).
- 문제 시나리오: vague class/relation instantiate, 응용에서 vague fact 사용(추천 시스템의 "comedy" 동의 여부), vague data 통합, vague model 재사용. **ML feature가 vague**하면(old/new, includesFamousActors) training data의 해석만 학습.

### 3.1 Vagueness Is a Feature, Not a Bug
> 모든 걸 crisp하게 정의해도 사용자가 그러리란 보장이 없다.

- 시나리오: "moderate price·exotic cuisine" 식당 검색(moderate=degree-vagueness, exotic=combinatory), RFP 평가(core competence·strong competitor·high budget 모두 vague), "moderately experienced Java developer" 채용(content에 vagueness). 엄격 경계는 €19.90 누락, borderline 양분도 누가 borderline인지 결정 필요.

### 3.2 Detecting and Describing Vagueness
> vagueness를 피할 수 없으면 최소한 사용자에게 경고해야 한다.

- 4단계: ① vague element 식별(수동/자동), ② 정말 vague인지 vs vague하게 정의됐을 뿐인지 조사, ③ type·dimension·context를 가능한 specific하게, ④ 문서에 명시.
- **간단한 vagueness detector**: WordNet의 vague/nonvague 형용사 sense 2,000개로 bag-of-words 분류기 학습(test accuracy 84%). CiTO 44 relation 적용 시 82% 정확(vague 74%, nonvague 94%). **vagueness ≠ subjectiveness**(OpinionFinder로 vague를 subjective로 보면 효과 없음 — vague relation 7% accuracy).
- **describing vagueness**: 실제 vague 여부, type(quantitative/degree vs qualitative/combinatory), quantitative의 dimension, provenance·applicability context(subjective·context-dependent). 예: isNearTo(distance degree), isCompetitorOf(business area·target market 2 dimension), belongsToCategory(combinatory). 목표는 vagueness 제거가 아니라 명시 — fuzzification·contextualization은 Ch12·13.
- **case study**(business process ontology): SUPER 프로젝트 ontology(BPMO, BGO, BROnt, BMO)에서 중심 element가 vague(hasBusinessGoal, Strategic Goal, Desired Result). tender call evaluation 과정에서 "high budget"·"adequate experience"·competitor가 vague → 다른 사람이 다른 결과 산출(측정·규칙 부족이 아닌 human thinking의 내재적 문제).

---

## 4. Not Documenting Biases and Assumptions
> 환경 제약에 따른 결정은 받아들일 수 있으나, 양심적으로 하고 모든 당사자에게 알려야 한다.

- 예: taxonomy에서 "tester"를 Software Tester에만 할당(app이 ambiguity 처리 못 함, 데이터상 90% 의미) — 선택 자체가 아니라 **미문서화**가 문제(비ambiguity 지침 모르는 신규 멤버가 다른 entity에 tester 추가).
- 흔히 미문서화되는 정보: constraint·restriction(multiple parenthood 금지, label 형식), context dependence(app X·user Y 전용), provenance(source·도구·전문가 수), quality(relation precision 85%, 프랑스어 coverage 25%), applicability(semantic search 최적화), design decision·bias(accuracy 우선, 특정 subdomain 편향).
- 이런 정보 부재는 usability 저하·실수 유발·유지 곤란.

---

## Summary (핵심 정리)
- context로 ambiguous·inaccurate name을 이해하리라 과신하지 말고 element 이름을 가능한 specific·clear하게 짓는다.
- textual definition을 경시하지 말고 흔치 않은 지식에 사용한다.
- vagueness를 무시하거나 flaw로 취급하지 말고, 탐지해 피할 수 없으면 type·dimension·context를 문서화한다.
- bias·가정·design decision을 양심적으로 다루고 모든 stakeholder에게 알린다.
