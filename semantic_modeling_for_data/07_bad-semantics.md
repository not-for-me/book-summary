# 07. Bad Semantics

## 챕터 개요 (3줄 요약)
- modeler가 modeling language의 element(subClassOf, sameAs 등)를 의도된 의미대로 쓰지 않아 부정확한 모델·잘못된 inference가 발생한다.
- 흔한 실수: bad synonymy·mapping(identity), instantiation/part/rigidity를 subclass로 오용, hierarchical·vague relation을 transitive로, inference rule을 constraint로 오인.
- semantic inference ≠ logical inference이며, 잘못된 입력은 잘못된 출력을 낳는다.

---

## 1. Bad Identity
> identity는 두 element가 같은 의미인지 판정하는 문제로, 잘못 다루면 erroneous inference를 낳는다.

### 1.1 Bad Synonymy
> 의미가 같지 않은 term을 synonym으로 정의하면 다른 의미가 완전 동등으로 취급된다.

- 예: ESCO에서 Economist의 label에 Interest Analyst·Labor Economist 혼재, Babelnet에서 Arsenal FC=Manchester United → search·assistant가 잘못 동작.
- 원인: ① term은 필요하나 새 entity 만들기 싫음(개발·유지 overhead, app 제약 — entity 5~6천 개 제한), ② synonymy를 search expansion similarity와 혼동(synonym ring에 Java~C++ 추가), ③ **false friends**(embarrassed/embarazada, sensible 영/불·서), ④ context dependence 무시(big/large는 size엔 OK 중요도엔 X), ⑤ 도메인 미묘한 차이(violin/fiddle, CEO/COO).
- 회피: synonymy는 "여러 context에서 의미 동등"임을 명확히(threshold 높게), 다수 judge + inter-agreement, synonymy/hyponymy/relatedness 차이 교육, **set으로 평가**(A-B, B-C, C-A 일관성), 기준·가정·bias 문서화·일관 적용, 확신 없으면 synonym이라 부르지 말 것(Textkernel은 hasAttractor 사용), 외부 model 재사용 시 주의.

### 1.2 Bad Mapping and Interlinking
> owl:sameAs로 다른 모델 element를 연결할 때 bad synonymy와 referential opacity 문제가 생긴다.

- owl:sameAs 오용 3종: 매우 유사하나 동일하지 않음, 특정 context에서만 동일, **referential opacity**(동일하지만 한 모델의 attribute가 다른 모델엔 부적합 — OpenCyc India를 WordNet synset에 연결 시 "NounSynset의 instance" 같은 무의미 assertion).
- **OWL/SKOS trap**: owl:sameAs ≠ owl:equivalentClass(전자는 동일, 후자는 extension 동등; sameAs는 class를 individual로 다뤄 OWL Full 필요). owl:sameAs ≠ skos:exactMatch(전자는 statement merge, 후자는 미merge).
- 회피: 연결 전 재고(Linked Data 비전이 business case는 아님 — Textkernel은 ESCO만 매핑, DBpedia는 안 함), 외부 model semantics 면밀 점검(Ch8), 선택적 import(ESCO label은 import 안 함), **skos:closeMatch 유혹 회피**(vagueness가 더 많은 오용 유발).

---

## 2. Bad Subclasses
> class subsumption(A⊂B면 A의 모든 instance가 B의 instance)이 자주 오용되어 problematic reasoning을 낳는다.

### 2.1 Instantiation as Subclassing
> entity를 type에 instantiation 대신 subclass로 연결하는 흔한 오류다.

- 예: KBpedia에서 Calgary가 City의 subclass, Parthenon이 Landmark의 subclass(개별 entity라 class 불가). SNOMED에서 Factory Worker가 Occupation의 subclass(→ 개별 사람이 occupation으로 추론됨).
- 원인: "is a"의 모호성(instantiation·subclass 둘 다), second-order class 미허용(OWL-DL — Human이 Species의 instance), 한 entity에 두 sense(democracy=system vs state), subclass 계층을 narrower/broader 계층과 동일시, 잘못된 가이드라인("concept=class", "individual instances are most specific concepts").
- 탐지: subclass의 instance가 superclass의 instance인지 확인, "is a kind of" pattern 적용("Human is a kind of Species"는 말 안 됨), **identity criteria 공유 확인**(Occupation ≠ Person), 정확한 naming(Democracy 대신 DemocraticCountry).

### 2.2 Parts as Subclasses
> part-whole를 subclass로 표현하는 오용이다(Engine을 Car의 subclass로).

- 원인: subclass=subset, subset=part 혼동(집합의 part vs 멤버의 part 구분), part-whole 계층을 class로만 표현 가능하다는 착각.
- 해결: part가 individual이면 part-of relation으로 직접 연결, class면 W3C 등의 복잡 pattern("Engine is part of Car"="모든 Car instance가 Engine instance를 part로 가짐"은 binary relation으로 불가).

### 2.3 Rigid Classes as Subclasses of Nonrigid Classes
> rigid class를 nonrigid class의 subclass로 두는 의미 오류다.

- Customer는 nonrigid(instance가 존재를 멈추지 않고 customer를 그만둘 수 있음), Person/Organization은 rigid → Customer를 superclass로 두면 Person instance가 본질적으로 Customer라는 잘못된 함의. Bacterium(rigid)과 InfectiveAgent(nonrigid)도 동일. rigid는 nonrigid의 subclass가 될 수 없음(nonrigidity 상속).
- 원인: role을 채우는 대안 class 표현에 subclassing 사용. subclass는 "가능한" instance가 아닌 "확실한" instance에 관한 것.

### 2.4 Common Superclasses with Incompatible Identity Criteria
> 두 disjoint superclass(호환 불가 identity)를 가진 class는 잘못이다.

- Customer가 Person·Organization의 subclass면 모든 Customer가 동시에 Person이자 Organization(disjoint라 불가). 다중 superclass면 identity 호환 확인, 아니면 상속 identity 수만큼 split(PrivateCustomer/CorporateCustomer).

---

## 3. Bad Axioms and Rules
> axiom·reasoning rule에서도 problematic modeling이 발생한다.

### 3.1 Defining Hierarchical Relations as Transitive
> 계층의 transitivity를 당연시하면 무의미한 assertion이 나온다.

- 유효: Cat⊂Mammal⊂Animal. 무효: Einstein은 Physicist, Physicist는 Profession이나 Einstein은 Profession 아님; Vehicles⊃Cars⊃Wheels이나 Vehicles⊅Wheels; Amsterdam∈Netherlands, Brazil∈UN이나 Amsterdam∉UN.
- 원인: 결합 불가한 다른 type의 relation 혼합, 단일 relation이라도 transitive 아닐 수 있음.
- **part-of**는 특히 문제: 6 type(Component-Integral, Member-Collection, Portion-Mass, Stuff-Object, Feature-Activity, Place-Area) 혼합 시 무효(musician's heart-orchestra). Member-Collection은 절대 비transitive, Component-Integral도 때때로 비transitive(house-door-handle).
- **SKOS**: skos:broader/narrower를 default transitive로 안 함(skos:broaderTransitive 별도 지원). 단 broaderTransitive가 broader의 super-relation이라 여전히 문제 가능 — 이름만 믿지 말고 정말 transitive일 때만 사용.

### 3.2 Defining Vague Relations as Transitive
> vague relation의 transitivity도 문제다.

- isNearTo(Italy~Greece~Turkey이나 Italy≁Turkey), client similarity(A~B revenue, B~C industry이나 A≁C). vagueness가 transitivity를 파괴(dimension·기준이 쌍마다 다름). SKOS도 skos:related는 비transitive라 경고.

### 3.3 Complementary Vague Classes
> vague class의 logical negation은 complement와 같지 않다.

- owl:complementOf(ChildlessPerson=Person ∩ not Parent). HappyPerson의 경우: HappyPerson과 NotHappyPerson이 정말 상호배타적인가?(borderline 때문에 not necessarily — happy 명시 안 됐다고 not happy 추론 부적절), NotHappyPerson과 UnhappyPerson이 같은가?(morphological antonym "unhappy"는 genuine antonym "sad"·negation "not happy"와 다름). complementary class naming이 vagueness 시 큰 차이.

### 3.4 Mistaking Inference Rules for Constraints
> RDF(S)/OWL의 domain·range는 constraint가 아닌 inference rule로 동작한다.

- wasBornIn의 domain=Person, range=Location일 때 "Amsterdam wasBornIn Ajax"라 하면 reasoner가 불평 않고 Amsterdam을 Person·Ajax를 Location으로 **추론**. OWA 설계 때문 — 모델 언어의 "fine print"를 읽어야.
- **SHACL**: W3C가 2017 도입 — RDF에 대한 constraint 검증과 OWL 미지원 inference rule 정의 가능.

---

## Summary (핵심 정리)
- modeling framework의 각 element가 어떻게 동작하도록 의도됐는지 반드시 이해한다.
- semantic inference ≠ logical inference — 잘못된 입력은 잘못된 출력을 낳는다.
- synonym·equivalence relation은 보이는 것만큼 단순하지 않으니 주의한다.
- subclassing을 남용하지 말 것 — 모든 concept을 class로, 모든 계층을 subclass로 모델링하지 않는다.
- 계층은 항상 transitive가 아니며, constraint로 위장한 inference에 주의한다.
