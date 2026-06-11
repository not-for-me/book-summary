# 02. Semantic Modeling Elements

## 챕터 개요 (3줄 요약)
- 서로 다른 community(DB, Semantic Web, linguistics)는 같은 semantic modeling element를 다른 terminology로 부르며, 이 차이를 알아야 모델 간 의미 오류 없이 mapping할 수 있다.
- 핵심 general element는 entity, relation, class/individual, attribute, axiom, term이며, 본서는 혼란을 피하기 위해 통일된 용어를 사용한다.
- 공통·표준화된 element로 lexicalization/synonymy, instantiation, subsumption, part-whole, relatedness, mapping, documentation 등이 있다.

---

## 1. General Elements
> 대부분 framework에 공통인 entity, relation, class, attribute, axiom, term을 통일된 용어로 정리한다.

### 1.1 Entities
> entity는 구체적(concrete, 시공간 내 존재·proper noun) 또는 추상적(abstract, idea/category/concept)으로 존재하는 것이며, 모델 내에서 unique·unambiguous해야 한다.

- 같은 entity가 여러 이름(Rome="Eternal City"), 같은 이름이 여러 entity(Rome, Georgia). "Things Not Strings"의 핵심.
- **abstract entity가 modeling이 더 어려움**: 보편 정의 합의 곤란(love, success, data scientist), 의미가 더 자주 변함(masculinity vs Battle of Waterloo). 생성자·소비자 모두 주의 — 예: film genre 해석 불일치 시 ML classifier precision 저하.

### 1.2 Relations
> relation은 둘 이상 entity가 관련되는 방식을 표현하며(binary/ternary), 기계 해석 가능한 의미·context를 제공한다.

- terminology 혼란: relational DB의 relation=table, E-R은 relationship, RDF(S)는 property, OWL은 object property, Description Logics는 role. 본서는 **relation**으로 통일.
- 고립된 entity는 이름·정의로 사람은 해석 가능하나 기계는 불가 — 예: "Java Developer is a kind of Software Developer" 관계를 알아야 기계가 이력서 적합성 판단.

### 1.3 Classes and Individuals
> class는 다른 entity의 semantic type 역할을 하는 추상 entity이고, 그 instance를 individual이라 한다.

- class는 같은 구조·특성을 공유하는 entity를 grouping하는 abstraction — 모든 instance에 적용되는 fact/rule 정의 가능.
- **"concept = class"는 부분적으로만 맞음**: concept이 instance를 함의하면 class(Song→특정 곡), 아니면 class 아님(Biology는 instance 없음). 유일 기준은 "instance가 될 수 있는 entity가 있는가". 이 fallacy는 Ch7의 여러 pitfall로 이어짐.
- framework별 차이: property graph는 class/instance 구분 없음(모두 node), SKOS는 Concept의 instance 못 만듦, OWL은 class 가능 entity를 individual로 모델링하기도. E-R(Chen)은 proper noun=individual, common noun=class. OWL-Lite/DL은 class와 individual 겸용 금지.
- Eagle, Data Scientist처럼 class·individual 양쪽으로 정당하게 모델링 가능한 경우의 dilemma는 Ch12에서 다룸.
- **ML class vs semantic class**: ML class는 분류 category(spam/nonspam), semantic class는 더 엄격해 instance가 있어야 함. Eiffel Tower/Parthenon은 ML class지만 semantic class는 아님(이미지는 monument가 아닌 Image의 instance).

### 1.4 Attributes
> attribute는 다른 entity와의 relation으로 표현하지 않고 literal value(숫자·문자열·날짜)로 표현하는 특성이다.

- 사용처: 별도 entity로 부적절한 특성(age, height, salary), 도메인 외 administrative 정보(추가자, 발견 방법).
- relation vs attribute 구분은 종종 불명확한 표현 선택(DBpedia의 filmAudioType은 attribute, campusType도 — 본질 차이 없음). 이 dilemma도 Ch12.
- relation attribute: relation 자체의 특성("John is friends with Sally since 2013"의 2013은 friendship의 attribute). RDF(S)는 property, OWL은 datatype property → 본서는 **attribute**로 통일.

### 1.5 Complex Axioms, Constraints, and Rules
> 일부 framework는 reasoning(명시되지 않은 fact 도출)을 가능하게 하는 복잡한 element를 제공한다.

- 예: E-R의 cardinality constraint, OWL의 relation range로 class 정의(Parent), SWRL rule(A의 parent B, A의 brother C → C는 B의 uncle).
- axiom은 framework마다 동작이 달라 undesired reasoning 유발 — **inference rule을 constraint로 오인**하는 경우 등(Ch7).
- semantic model의 reasoning은 주로 **deductive**(전제 참+논리 따르면 결론 필연적 참). 그 외 **abduction**(결론→전제 역추론, ML 설명에 활용)과 **induction**(전제·결론→rule 추론, supervised ML 학습) — 둘 다 항상 참은 아님. 설계·사용 시 필요한 reasoning 종류를 명확히 해야.

### 1.6 Terms
> term은 element를 lexically 기술하는 문자열로, 그 자체로 의미는 없고 element의 의미 일부를 표현한다.

- 한 term이 여러 의미(bank), 다른 표현이 같은 entity(Rome="Eternal City"). term만으로는 의미를 표현 못 하므로 semantic model이 될 수 없음.
- taxonomy의 term도 사실은 함의된 의미를 가진 entity/concept(bank=금융기관). WordNet은 단어를 의미 단위 **synset**(sense)로 grouping(code는 명사 3 sense, 동사 2 sense).
- semantic modeling의 본질 = word/term을 sense에 연결하는 것.

---

## 2. Common and Standardized Elements
> 대부분 framework에 나타나는 표준화된 element들 — 이름은 제각각이다.

### 2.1 Lexicalization and Synonymy
> lexicalization은 element를 자연어 term(synonym·lexical variant)에 연결하는 관계다.

- **synonym**: 넓은 context에서 의미가 (거의) 같은 다른 term(cats/felines, aspirin/acetylsalicylic acid, tissues/Kleenex, lifts/elevators). 진짜 synonymy는 드묾.
- **lexical variant**: 같은 term의 다른 word form(spelling·문법·약어 — Romania/Rumania, mice/mouse, PVC).
- framework별: RDF(S)/OWL은 `rdfs:label`, SKOS는 preferred/alternative/hidden label, WordNet은 lemma, ANSI/NISO는 preferred(descriptor)/nonpreferred term.
- 중요성: 의미를 더 명확히 하고, data science app이 언어적 다양성("software developer"="software programmer")을 더 정확히 처리. 단 의미 다른 synonym 포함은 pitfall(Ch7), 모든 lexicalization 포함이 늘 좋은 건 아님(Ch13).

### 2.2 Instantiation
> entity를 그것이 속한 class(들)에 연결하는 관계다.

- RDF(S)/OWL의 `rdf:type`, ANSI/NISO의 BTI/NTI. entity가 가질 특성·behavior를 class로부터 알게 하고, class의 의미를 instance로 명확히 함.

### 2.3 Meaning Inclusion and Class/Relation Subsumption
> 한 element의 의미가 다른 element 의미에 포함되는 관계(linguistics의 hyponymy/hypernymy)다.

- class에 적용 시 **subsumption(subclassing)**: A가 B의 subclass면 A의 모든 instance는 B의 instance(Soccer Game⊂Sports Event). relation에도 적용(hasFather⊂hasParent).
- RDF(S)/OWL의 `rdfs:subClassOf`, `rdfs:subPropertyOf`. 잘못 모델링되어 problematic inference 유발하는 경우 많음(Ch7).

### 2.4 Part-Whole Relation
> part-of로 표현되는 관계(linguistics의 meronymy/holonymy), ANSI/NISO의 BTP/NTP다.

- 논리적으론 하나지만 의미적으론 6변형: component-integral(wheel-car), member-collection(person-crowd), portion-mass(slice-cake), stuff-object(building-steel), feature-activity(testing-software development), place-area(Manhattan-NYC). 잘못 모델링 시 wrong inference(Ch7).
- **Hierarchical relation**(taxonomy): instantiation·meaning inclusion·part-whole를 포괄. SKOS `skos:broader/narrower`, ANSI/NISO BT/NT.

### 2.5 Semantic Relatedness
> 정확한 성격을 명시하지 않고 두 element 의미가 관련됨을 나타내는 관계(associative/semantic similarity)다.

- SKOS `skos:related`, ANSI/NISO RT. context 밖에선 유사성 합의가 쉬우나 context 내에선 어려움(Ch10) — 특정 context의 모델 개발·사용 시 relatedness 적절성에 주의.

### 2.6 Mapping and Interlinking Relations
> 서로 다른 모델의 element를 연결해 semantic interoperability를 가능케 한다.

- OWL `owl:sameAs`(동일 individual, ML=Apprentissage_automatique), `owl:equivalentClass`(extensional 동등, 항상 같은 instance).
- SKOS 5종: exactMatch(높은 확신 호환), closeMatch(일부 app 호환), broadMatch, narrowMatch, relatedMatch.
- interlinking은 error-prone — sameAs가 잘못 연결되는 경우 많음(Ch7). 사용 시 품질 점검 필수, 매핑 개발·유지 비용 가치 판단은 Ch13.

### 2.7 Documentation Elements
> 형식적 정의 외에, 사람이 읽을 수 있는 정의·provenance·scope note도 모델 이해·유지·사용에 매우 유용하다(Ch6).

- **Definitions and examples**: 4종 — extensional(외연 나열, 유한·소규모만), intensional(필요충분조건, bachelor="unmarried man", 무한 집합에도), genus and difference(상위 category+구별 특성, miniskirt), ostensive(예시 지시, 색·감각). RDF(S)/OWL은 `rdfs:comment`, SKOS는 `skos:definition/example`.
- **Scope and usage**: 이름·정의로 안 드러나는 의도·적용 정보(제외된 다른 의미, 특정 context 부적용, 특정 task 최적화, 알려진 bias).
- **History and provenance**: 변경(who/when/what)·version·source 추적. SKOS historyNote/editorialNote/changeNote, OWL versionInfo/priorVersion/deprecated, 또는 PROV-O ontology.

---

## Summary (핵심 정리)
- semantic modeling은 표준 용어가 없어 community마다 같은 것을 다르게(또는 다른 것을 같게) 부른다 — 이름만으로 element를 이해하지 말고 문서를 읽어야 한다.
- 핵심 element: entity(특히 abstract entity 주의), relation, class/individual, attribute, axiom, term.
- semantic class ≠ ML class(ML class는 semantic model에선 individual일 수 있음), class ≠ concept, subclass 계층 ≠ broader/narrower 계층.
- 공통 element로 lexicalization/synonymy, instantiation, subsumption, part-whole(6변형), relatedness, mapping(error-prone), documentation이 있다.
- semantic model은 주로 deductive reasoning을 위한 것이며 abduction·induction은 보조적이다.
