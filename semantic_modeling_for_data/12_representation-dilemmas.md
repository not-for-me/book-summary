# 12. Representation Dilemmas

## 챕터 개요 (3줄 요약)
- 같은 정보를 같은 언어 안에서도 여러 방식으로 표현할 수 있으며(class vs individual, subclass 여부, attribute vs relation), 각 방식은 강약점이 다르다.
- 핵심 판단 기준은 entity의 instance·관계·attribute를 얼마나 semantically 기술하고 싶은가이다.
- vagueness는 fuzzification(0~1 truth degree)으로 machine-processable하게 표현할 수 있으나, disagreement를 줄이고 비용을 상회할 때만 의미 있다.

---

## 1. Class or Individual?
> class·individual을 엄격 구분하는 framework에서 둘 다로 모델링 가능한 entity를 어떻게 표현할지의 dilemma다.

- 문제: OWL-DL은 class를 다른 class의 instance로 못 만들고 class 간 직접 relation 제한(John∈Data Scientist∈Occupation 동시 표현 불가).
- 판단 3질문: ① instance가 있나?(없으면 class 아님), ② 그 instance를 기술·관계 정의하고 싶나?(게임 사본까지?), ③ class로 모델링하면 표현 못 하는 fact가 있나?(hasEssentialSkill, averageSalary).
- 셋 다 positive면 어려움 → workaround: ① individual로 + custom relation(hasProfession, belongsToSpecies — 자유롭지만 class axiom 못 씀), ② class·individual 두 entity로 다른 이름(Data Scientist (Profession)/(Professional) — 거의 모두 표현 가능하나 둘 연결·sync 어려움), ③ OWL2 **punning**(같은 식별자를 context로 class/individual 판정 — 단 metaclass의 실제 해결이 아닌 syntactic trick, 혼란 주의).

---

## 2. To Subclass or Not to Subclass?
> class를 specialize할 때 새 subclass로 할지 relation/attribute 값으로 할지의 dilemma다.

- 예: Restaurant를 cuisine별로 — ① subclass(AsianRestaurant 등, 간단·reasoning 활용하나 cuisine 자체를 말하기 어려움), ② Cuisine class + hasCuisine relation(cuisine을 말할 수 있으나 subclass 편의 상실).
- class vs individual과 같은 trade-off + 추가 질문: ① subclass가 parent·sibling과 다른 추가 attribute·relation·restriction을 갖나?(mammal은 vertebrate, 왼/오른 폐는 차이 없음), ② subclass가 도메인에서 흔히 쓰이나?(comedy film은 자주, 더빙 언어로는 드묾), ③ subclass의 rigidity는?(AsianRestaurant은 비교적 안정, RestaurantWith50Employees는 volatile — 저rigidity는 유지 overhead 재고).
- ambiguity 주의: 짧은 이름(TarantinoFilm) 유혹 — 보통 OK지만 가정 범위 주의.

---

## 3. Attribute or Relation?
> attribute와 relation 사이엔 엄격한 개념적 구분이 없고, 값을 얼마나 semantically 기술하고 싶은가가 관건이다.

- 예: DBpedia는 film의 originalLanguage를 relation(Language class 연결, iso6391Code 등)으로, filmColourType을 attribute(문자열)로. 언어 semantics가 중요 없고 color type을 풍부히 기술하고 싶으면 반대로.
- 숫자·날짜는 거의 항상 attribute(무한 값을 entity로 만들면 비실용).

---

## 4. To Fuzzify or Not to Fuzzify?
> vagueness를 0~1 truth degree로 machine-processable하게 표현하는 fuzzification 기법이다.

- Zadeh(1960s)의 개념: vague statement에 0(거짓)~1(참) 실수 부여("John∈YoungPerson to degree 0.8"). **fuzzy degree ≠ probability**(probability는 truth condition이 명확한 사건의 likelihood, fuzzy는 truth condition이 미정인 것의 perceived 정도; probability theory vs fuzzy logic). 둘 다 underlying 현상(uncertainty/vagueness)을 줄이려 함.

### 4.1 What Fuzzification Involves
> vague element 탐지·분석 → fuzzify 방법 결정 → degree 수집 → 품질 평가 → degree 표현 → 적용의 6단계다.

- **Fuzzification options**: ① 1차원 quantitative → fuzzy membership function(age→degree, trapezoidal/triangular/shoulder/linear). 30세는 Young 0.5·MiddleAged 0.5. ② 다차원 → multivariate function 또는 차원별 function을 **t-norm/t-conorm**으로 결합(min/Gödel, product, Łukasiewicz). 최적 t-norm 규칙 없어 실험 필요. ③ statement당 직접 degree(차원 너무 많거나 qualitative일 때, 단 확장성 떨어짐).
- **Harvesting**: 불일치를 포착·정량화 — 직접 질문(구조·통제, 비확장적; vagueness는 noncomplementary라 "short?" 묻지 말 것), 간접 질문(app 맥락, 추천 설명에 동의? — 자연·확장적이나 통제 어려움), data mining(리뷰·토론 — 자동·확장적이나 noise 많음).
- **Fuzzy model quality**: 올바른 element를 fuzzify했나(borderline case 있나), degree가 accurate한가(golden 값이 아닌 직관적으로 자연스러운가 — fuzzy distance), consistent한가, provenance 문서화됐나. **fuzzification bias 주의**(나이 "old" degree가 보험료 결정).
- **Representing**: 대부분 언어가 fuzzy 미지원(OWL·E-R fuzzy 확장은 학술적). 단순하면 자체 확장(truth degree relation attribute; relation을 entity로 모델링), 복잡 membership function·rule은 기존 framework 재사용.
- **Applying**: app 개발자와 협업해 truth degree를 활용하게 — semantic tagging·disambiguation·search·match에 유효.

### 4.2 When to Fuzzify
> degree 획득·유지의 난이도·비용 때문에 신중해야 하며, 6질문으로 판단한다.

- 질문: ① 어떤 element가 불가피하게 vague?, ② 불일치가 얼마나 심각·영향?, ③ vagueness가 원인인가 다른 요인인가?, ④ degree가 불일치를 줄일까?, ⑤ app이 degree를 활용·이득 볼 수 있나?, ⑥ 이득보다 싸게 확장적으로 degree 획득·유지 가능한가?
- 불일치 측정(Cohen's Kappa), 영향은 실사용 모니터링. ④⑤는 **A/B testing**(fuzzy vs 비fuzzy 버전 비교). ⑥이 가장 중요. **fuzzification은 vagueness 문서화의 면제가 아니라 오히려 더 필요**.

### 4.3 Two Fuzzification Stories
> fuzzification이 app 효과를 향상시킨 두 사례.

- **Fuzzy electricity**(2008 그리스 전력시장): vague relation 3개(isImportantPartOfProcess 등)를 전문가 불일치를 degree로 변환(0.5, 0.9 등). fuzzy 버전이 search precision·recall 약 7% 향상.
- **Fuzzy actors/warriors**(Knowledge Tagger thematic scope resolution): 영화 리뷰 — crisp hasPlayedInFilm 대신 fuzzy wasAnImportantActorInFilm("Robert Duvall in Apocalypse Now 0.6") → 100 리뷰에서 accuracy 10% 향상. military conflict — DBpedia 기반 fuzzy ontology → accuracy 13% 향상.

---

## Summary (핵심 정리)
- 유용한 subclass는 parent와 다른 추가 attribute·relation을 갖고 도메인에서 흔히 쓰이는 것이다.
- 저rigidity subclass는 유지가 어려우니 피한다.
- 특성 값의 의미를 semantically 기술하고 싶으면 attribute 대신 relation으로 표현한다.
- fuzzy degree는 probability가 아니다.
- fuzzification은 vague statement의 불일치를 줄이고 그 이득이 비용을 상회할 때만 의미 있으며, vagueness 문서화 필요를 오히려 늘린다.
