# 03. Semantic and Linguistic Phenomena

## 챕터 개요 (3줄 요약)
- 본서의 pitfall·dilemma 다수는 human language의 ambiguity, vagueness, uncertainty, semantic change 같은 현상에서 비롯된다.
- 이 현상들의 정확한 성격을 이해하는 것이 pitfall·dilemma를 해결하는 첫걸음이다.
- rigidity/identity/unity/dependence, symmetry/transitivity, closed/open-world assumption은 reasoning 정확성에 직결된다.

---

## 1. Ambiguity
> ambiguity는 하나의 정보가 둘 이상으로 그럴듯하게 해석될 수 있는 상황이다.

- 유형: phonological("ice cream"/"I scream"), syntactic(문장 구조), anaphoric(대명사 지시 대상), **term-level(lexical)**(단어 다의 — Kashmir=곡/지역), sentence-level("John and Jane are married"=서로/각자).
- semantic model과 함께 언급되는 두 상황: ① ambiguity 있는 source로 model 개발 → 각 element가 input data의 의미를 unambiguous하게 표현하도록(John/Jane 신원 확정, isMarriedTo 관계 vs marital status attribute — Ch6), ② model을 data disambiguation system(예: YAGO2 기반 AIDA의 NER)에 사용 → model이 task에 최적화돼야(Ch10).

---

## 2. Uncertainty
> uncertainty는 필요한 지식의 부재로 statement의 진위를 판정할 수 없는 현상이다.

- **explicit**(probably, might, perhaps 등 keyword) / **implicit**(확언하나 source·acquisition을 신뢰할 이유 부족 — false statement 다수 source, <100% precision 추출 system).
- 개발 시: uncertainty의 성격·provenance·수준을 사용자에게 알려야(모델 전체/일부/개별 statement). 사용 시: 개발 조건을 파악해 신뢰 정도 추정.

---

## 3. Vagueness
> vagueness는 borderline case를 허용하는(또는 sharp boundary가 없는) predicate에서 나타나는 현상이다.

- **Sorites Paradox**(밀알 몇 개부터 heap?): 추론은 멀쩡하나 결론이 거짓 — heap 정의에 최소 개수가 없는 게 문제. 그 경계 수를 정하기 어려운 것이 vagueness.
- 2종: **degree-vagueness**(차원상 경계 부재 — bald는 머리카락 양, tall은 키), **combinatory vagueness**(여러 조건의 필요충분 조합을 명확히 못 가름 — Religion).
- inexactness("170~180cm", 경계는 정확), ambiguity, uncertainty와 **구별**할 것. **context-dependent**(tall은 평균 인구 vs 농구선수에 따라 다름).
- model에서의 위치: vague class(phase/state — Adult, attribution — TallPerson), vague relation(hasGenre, hasIdeology), vague attribute value(cheap/moderate/expensive 등 gradable). 예: CiTO의 plagiarizes·citesAsAuthority·supports는 vague, sharesAuthorInstitutionWith·retracts는 crisp.
- vague element를 crisp처럼 다루면(ESCO의 essential) 품질·usability 저하. 처리법은 Ch6·7·13에서 — liability를 asset으로 전환.

---

## 4. Rigidity, Identity, Unity, and Dependence
> OntoClean의 네 ontological 개념으로 class subsumption의 의미적 정확성을 검증한다.

- **Rigidity**(essence 기반): 모든 instance에 본질적이면 **rigid**(Human), 본질적이지 않으면 **anti-rigid**(Student, Food), 일부에만 본질적이면 **semi-rigid**(HardThing — hammer엔 본질, sponge엔 아님).
- **Identity**: 두 entity가 같은지 판정 기준. Person은 identity 있음(DNA), RedThing은 없음(둘 다 빨갛다고 동일 아님). membership criterion(class 소속 판정)과 다름.
- **Unity**: instance가 whole entity인지. Ocean은 unity 있음(Pacific Ocean), AmountOfWater는 없음(one gallon). unity/non-unity(LegalAgent — 사람·조직 다른 기준)/anti-unity.
- **Dependence**: C1의 모든 instance에 C2 instance가 있어야 하면 C1은 C2에 dependent(Food — 먹는 대상이 있어야 food). 이 네 개념 적용으로 subsumption 오류 회피(Ch7).

---

## 5. Symmetry, Inversion, and Transitivity
> relation에 관한 현상들로, reasoning system 구축에 결정적이다.

- **symmetric**: A R B → B R A(cousin). **transitive**: A R B, B R C → A R C(located in). **inverse**: A R1 B → B R2 A(brother/sister).
- modeler는 이를 정확히 인식·포함해야 하나, 겉보기엔 이 속성을 가진 듯해도 실제론 없는 경우가 있음(Ch7).

---

## 6. Closed- and Open-World Assumptions
> CWA는 모르는 statement를 거짓으로 추론하고, OWA는 결론을 내리지 않는다.

- **CWA**: 진위 모르면 거짓(seat 미배정 → 미check-in). 정보 입력을 완전 통제·consistency 강제 가능할 때 타당.
- **OWA**: 진위 모르면 결론 없음(의료기록에 알레르기 미언급 ≠ 알레르기 없음). 완전성 보장 못 할 때 타당.
- 개발 시 어느 가정을 지원하는지 명확히 해 맞는 framework 선택. **RDF(S)/OWL은 OWA** — 다른 언어에서 constraint로 동작하는 axiom이 여기선 inference rule로 동작 → 모르면 undesired reasoning(Ch7).

---

## 7. Semantic Change
> semantic change(drift)는 term의 의미·용법이 시간에 따라 변하는 현상이다.

- 예: awful(경외→매우 나쁨), egregious(매우 좋음→매우 나쁨), demagogue, sly, prestigious, matrix.
- 형태: specialization(skyline), generalization(hoover, Google), metaphor(broadcast), metonymy(horn), synecdoche(수도→국가), hyperbole, meiosis, auto-antonymy(cleave), folk-etymology(contredanse), antiphrasis.
- **주의**: 특정 집단 advocacy로 인한 변화(gay, "manic depression"→bipolar disorder)는 offense·dispute 유발 가능.
- 측정: entity 표현의 세 측면 — labels(term), intension(attribute·relation), extension(instance 집합). extension의 역할은 entity 종류에 따라 논쟁적(Person은 extension 변해도 핵심 의미 불변, European Union은 회원국이 부분 정의).
- **terminology clash**: semantic modeling의 concept drift ≠ ML의 concept drift(target 변수 통계 속성 변화).
- 사용 시: model 의미가 data 의미와 정렬됐는지 확인(15세기 텍스트를 현대 ontology로 분석하지 말 것). 개발 시: 빠르게 변하면 항상 최신화 메커니즘 필요(Ch14).

---

## Summary (핵심 정리)
- vagueness, inexactness, ambiguity, uncertainty는 서로 다른 현상으로 다른 방식으로 다뤄야 한다.
- model 생성 시 모든 element를 unambiguous하게 기술하고, uncertainty의 성격·provenance·수준을 알려야 한다.
- rigidity/identity/unity/dependence는 subsumption 정확성, symmetry/transitivity·CWA/OWA는 reasoning 정확성에 직결된다.
- RDF(S)/OWL은 OWA를 따르므로 axiom이 inference rule로 동작함을 알아야 한다.
- semantic change의 성격·속도를 알아야 model 갱신 주기를 추정할 수 있다.
