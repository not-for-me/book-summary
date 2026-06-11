# 08. Bad Model Specification and Knowledge Acquisition

## 챕터 개요 (3줄 요약)
- requirement를 부정확하게 specify하면 비싼데 쓸모없는 model을 만들게 되며, 핵심은 context 이해 → feature 명세 → feasibility·importance 평가의 3단계다.
- 올바른 specification이라도 wrong knowledge source(data·people)나 wrong acquisition method/tool을 쓰면 model이 망가진다.
- data·people을 맹신하지 말고 systematic error·bias를 scrutinize하며, semantics를 먼저 명세한 뒤 source·방법을 고른다.

---

## 1. Building the Wrong Thing
> requirement를 잘못 명세하면 아무도 못 쓰는 것을 만든다(저자의 Textkernel 실패담).

### 1.1 Why We Get Bad Specifications
> 저자의 실수: 요구를 정확히 안 덮음, 중요 feature 누락, harmful feature 추가, 아무도 못 쓸 feature 추가.

- 원인: requirement gathering에서 clarity·specificity 미강조("synonym" 원했으나 사실은 search expansion 유사어), application과 함께 명세 안 함(도메인·data에만 집중), conflicting requirement 미예상, legacy·history 무시(과거 결함이 사실은 의도된 결정), pain point 오해(과거 프로젝트 편향).
- 표현 vs 실제: "legal 도메인"→"US legal만", "Spanish lexicalize"→"스페인의 모든 언어", "inference"→"inductive inference".

### 1.2 How to Get the Right Specifications
> patience·humbleness·inquisitiveness가 필요한 iterative 과정 — context 이해 → feature 명세 → feasibility·importance 평가.

- **Investigating context**: ① 어떤 app이 쓰나(가장 중요한 stakeholder, 단일 app이 이상적이나 미래 영향도 레이더에), ② 현재 어떤 model을 어떻게 쓰나(폐기 말고 history·원칙·강약점 이해), ③ 각 system의 pain point(focus — recognition vs disambiguation 중 무엇), ④ 과거 해결 시도·실패 원인(history 반복 방지). **사람들의 terminology를 해독**.
- **Specifying features**: 적정 generality의 **core entity types**부터(Athlete/Actor, Person 아님). **competency questions**(자연어 질문 — "What is the average salary of a data scientist in the US?" → 국가별 salary 필요)로 비전문가 stakeholder가 요구를 표현. premature modeling 주의, 모호한 질문은 명확화("in demand" 정의? Europe=대륙/EU? granularity?).
- **Assessing feasibility and importance**: feasibility는 ① modeling language 제약(구조·reasoning), ② acquisition resource 가용성·품질·비용(내용·expressiveness). importance는 전략·business 의존(stakeholder만 줄 수 있음). hype 주의("a" model은 쉬워도 "원하는" model은 어려움). importance가 quality strategy 주도, "low hanging fruit의 폭정" 주의(쉬운 것 우선이 technical debt 안 만들 때만 OK).

---

## 2. Bad Knowledge Acquisition
> 요구가 잘 명세돼도 wrong source·method를 쓰면 model이 망가진다.

### 2.1 Wrong Knowledge Sources
> data와 people이 source이며, 둘 다 부정확·부적절할 수 있다.

- data 4용도: as-is 재사용, 추출 대상, (semi-)supervised training, distant supervision. people 2용도: element 제공·curate·maintain·validate, training data annotate.
- **When data is wrong**: ① 의미적으로 부정확(Wikipedia/DBpedia/SNOMED 오류 전이), ② 정확하나 필요한 게 아님(CV에서 정의 찾기, supply data로 demand 채우기), ③ 정확·부합하나 추출 어려움(저구조·고ambiguity).
- data 오용 원인: 신뢰성 없는 data 맹신(creator 신용·문서만 보고 — ESCO도 accuracy·coverage 수치 없음), semantics 부정확 명세, **"because there is much more light out here" trap**(Nasreddin 우화 — 가진 data라 무조건 사용).
- 회피: semantics를 **먼저** 명세한 뒤 data 선택, correctness·compatibility·추출 용이성 scrutinize. **systematic error 탐색**(DBpedia가 Machine Learning을 Disease의 instance로 — 대부분 abstract entity가 잘못된 class).
- **When people is wrong**: 전문가도 부정확(ESCO의 data scientist essential skill "RDF Query Language"는 LinkedIn에 거의 없음). 전문가가 오히려 없는 causality를 봄(Aroyo·Welty). 원인: ① 필요한 expertise 없는데 인식 못 함(expertise는 단일 skill 아님), ② 본인이 모름(Dunning-Kruger, overconfidence), ③ cognitive bias(availability·groupthink·confirmation·anchoring 등). 회피: semantics 명세 후 정확한 expertise 식별, 다양·독립 viewpoint, 전문가 scrutinize(test session으로 systematic error·bias 관찰).

### 2.2 Wrong Acquisition Methods and Tools
> data·people 품질이 높아도 wrong method/tool을 쓰면 망가진다.

- **Word2Vec 오해**: "synonym 추출? Word2Vec 쓰면 됨"이 흔한 오답 — synonym만이 아닌 antonym·hyponym도 줌, term sense 구분 못 함(Apple 과일/회사 단일 vector). **Maslow's hammer**("망치만 있으면 모두 못으로 보임").
- 도구 부정확 문서화: Shalaby의 entity recognition이 논문엔 94% precision·96% recall이나 저자 적용 시 60~75%(test set에 Unknown category 없는 방법론 결함). **도구 효과 점수는 task·data 특성에 민감**(AIDA 83%→62%, Spotlight 81%→56%→34%) → **자기 data로 테스트** 필수.
- 도구 평가 7질문: 정확히 어떤 task·semantics 지원? 약속한 semantics 전달? bias 회피? 어떤 조건에서 어떤 effectiveness 범위? 사용 recipe? troubleshoot·optimize 방법? operationalize·maintain 방법?
- **Scrutinizing FRED** 사례: 유사 입력에 다른 출력(Greek history=individual, Greek literature=class), is-a를 거의 항상 instance-class로, 잘못된 class로 artificial entity 생성, class 편향, 불필요한 복잡성 → 사용 안 하기로 결정.
- **Failing humans-in-the-loop**: 모호·부정확한 질문("Is A a B?" → "Are all A's instances also instances of B?"), guideline 부족(특히 vagueness), **consensus 강요**(disagreement는 noise가 아닌 signal), viewpoint 부족, priority 부족, "직접 안 해봄"(annotation process를 본인이 먼저 해봐야).

---

## 3. A Specification and Knowledge Acquisition Story
> aspect-based sentiment analysis(ABSA)용 model 개발 사례(2015).

### 3.1 Model Specification and Design
> 3 competency question으로 명세 — 어떤 entity가 aspect인가, 어떤 evaluation expression이 쓰이나, aspect별 polarity는?

- SKOS 기반 model: **AspectEntity**(opinion 표현 가능 특성, skos:Concept의 subclass로 일반화 + taxonomy), **AspectEvaluation**(evaluation expression + polarity), hasAspectEvaluation relation, hasEvaluationExpression·hasPolarity attribute. 예: Food→TastyFood(positive)/DecentFood(neutral). rule: aspect A의 evaluation E를 narrower aspect가 상속.

### 3.2 Model Population
> 알려진 aspect의 evaluation 발견 + 미지 aspect-evaluation-polarity triple 발견을 위한 semi-automatic pipeline.

- pipeline: opinion 코퍼스 → subjectivity detection(OpinionFinder)으로 필터 → named entity resolution으로 known aspect 식별 → relation extraction(pattern 기반, dependency grammar)으로 aspect-evaluation 쌍 추출(tf-idf로 ranking) → polarity 결정(positive aspect는 positive context에 출현 가정, context sentiment 평균).
- **평가**(restaurant 도메인, 2,000 리뷰): known aspect precision 80%, unknown 72%(recall 낮음). threshold로 precision/recall 조절. polarity 정확도 80%(56 쌍, 수동 vs 자동 비교).

---

## Summary (핵심 정리)
- requirement gathering에서 clarity·specificity를 강제하고, 도메인·data뿐 아니라 모든 의도된 application에 중심 역할을 준다.
- biased·conflicting requirement를 예상하고, legacy·history를 무시하지 않으며, feasibility·importance를 항상 평가한다.
- hype를 경계 — 좋은 model 구축은 대개 어렵다.
- semantics를 먼저 명세한 뒤 knowledge source·method를 고르고, 신용을 넘어 scrutinize한다.
- bias를 인식하고 중화하거나 투명하게 만든다.
