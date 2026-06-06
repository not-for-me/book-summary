# 2. Organizing Principles for Building Knowledge Graphs

## 챕터 개요 (3줄 요약)
- 일반 그래프를 "지식그래프"로 만드는 것은 데이터에 의미를 부여하는 "조직 원리(Organizing Principle)"이며, 이는 사람·소프트웨어 모두를 위한 "계약(Contract)" 역할을 한다.
- 조직 원리는 단계적으로 강해진다: Plain Graph → Property Graph → Taxonomy(계층) → Ontology(다층 관계). 각 단계가 더 풍부한 추론을 가능케 한다.
- 핵심 설계 원칙은 "just-enough semantics"(필요한 만큼의 의미만) — 오버엔지니어링을 피하고 점진적·반복적으로 모델을 진화시켜라.

## Plain Old Graphs
> 조직 원리가 없는 일반 그래프는 데이터 해석 지식이 데이터가 아니라 "쿼리/프로그램 로직 속에 숨겨져" 있다.

- 예: 온라인 상점의 C(고객)-P(상품) 구매 그래프. 엔지니어가 의미를 코드에 인코딩해야만 "이 고객이 산 상품"을 답할 수 있다.
- 문제점: 도메인 지식 없는 데이터 과학자는 해석 불가, 그래프 제작자가 퇴사하면 알고리즘 역공학 필요.
- 해결책: 의미를 데이터 자체에 인코딩(조직 원리 적용)하면 잠재 지식이 표면화되어 지식그래프가 된다.

## Richer Graph Models (Property Graph)
> 속성 그래프 모델은 레이블·관계 타입/방향·속성을 제공해 그래프를 (어느 정도) "자기 기술적(self-describing)"으로 만드는 1차 조직 원리이다.

- 도메인 지식 없이도 가능한 처리: 레이블로 동일 엔터티 추출, 스키마 introspection, 시각화(Bloom, Linkurious).
- "조직 원리는 계약": 그래프와 소비 소프트웨어가 동일한 규약(레이블+방향성 관계+속성)을 지키면 도메인 지식 없이도 작동한다.
- 단, 속성 그래프는 저수준 원리 — 레이블만으로는 "대체 가능성(substitutability)" 같은 추론이 어렵다.

## Knowledge Graphs Using Taxonomies for Hierarchy
> 택소노미(Taxonomy)는 broader-narrower(일반-특수) 계층으로 "x는 y의 일종"이라는 추론을 가능케 하는 분류 체계이다.

- LPG에서 레이블은 "역할 태그"일 뿐 연관성(associativity)이 없다 → apple이 fruit의 하위라는 추론 불가. (이는 의도된 설계: 타입 시스템은 애플리케이션 관심사)
- Category 노드를 SUBCATEGORY_OF (또는 NARROWER_THAN/SUBCLASS_OF) 관계로 연결해 계층 구축, 상품을 적절한 위치에 연결.
- 분류는 동적(dynamic): 같은 상품을 Black Friday 반값 / 무선 헤드폰 / 고급 오디오 등 여러 계층에 동시 소속시킬 수 있다(그냥 노드·관계 추가).
- 택소노미가 있으면 path similarity, Leacock-Chodorow, Wu-Palmer 같은 표준 의미 유사도 알고리즘을 도메인 무관하게 적용 가능.

## Knowledge Graphs Using Ontologies for Multilevel Relationships
> 온톨로지(Ontology)는 계층을 넘어 PART_OF, COMPATIBLE_WITH, DEPENDS_ON, UPSELL 같은 풍부한 횡적 관계와 관계의 성질(transitive, symmetric)까지 표현한다.

- 수직(계층)뿐 아니라 수평(cross-cutting) 추론 가능: iPhone 12는 iOS 기기이므로 mobile phone 검색 결과로 유효, UPSELL 관계로 iPhone 12 Pro 추천.
- 시스템 간 "의미 다리(semantic bridge)" 역할: cross-equivalence 정의로 서로 다른 시스템의 동일 개념을 연결.
- 모듈식·계층식으로 구성하면 복잡도 관리 가능 — 각 레이어를 독립 쿼리하거나 합쳐서 교차 도메인 추론.
- 온톨로지가 지식을 "실행 가능(actionable)"하게 만든다: 재고 데이터와 연결해 품절 시 대체품/고마진 상품 추천.

> [모델링 관점 - 주식시장 도메인 적용]
> 이 장은 책의 모델링 철학의 핵심이다. 주식시장 지식그래프 설계 시: (1) Property Graph 레벨 — Company/Investor/Exchange/Sector 노드. (2) Taxonomy 레벨 — 산업 분류(GICS처럼 Sector > Industry Group > Industry > Sub-Industry)를 SUBCATEGORY_OF로 구축해 "동종업계" 추론. (3) Ontology 레벨 — COMPETES_WITH, SUPPLIES, HAS_EXPOSURE_TO(원자재/금리), CORRELATED_WITH, REGULATED_BY 같은 횡적 관계로 "이 기업 악재가 어느 공급망/섹터로 전파되는가"를 추론. 표준 온톨로지 FIBO(Financial Industry Business Ontology)가 금융 도메인 표준으로 명시되어 있으므로, 처음부터 직접 만들기보다 FIBO를 채택/확장하는 것이 상호운용성과 규제 보고 측면에서 유리하다.

## Which Is the Best Organizing Principle? / Standards vs Own
> 조직 원리는 "의도된 용도"가 결정한다. 거창한 메타모델을 미리 만드는 것("바다를 끓이지 마라")은 흔한 실수다.

- just-enough semantics: 현재 유스케이스에 필요한 의미만 도입하고 수요에 따라 추가. 단순 택소노미로 충분하면 복잡한 온톨로지를 만들지 마라.
- 점진적·반복(Agile)적 구축이 가치를 빨리 내고 리스크를 줄이며 "온톨로지 완벽주의"의 함정을 피한다.
- 표준 온톨로지 예: SNOMED CT(임상), LCC(학술), **FIBO(금융/비즈니스)**, Schema.org(웹), DCMI. 표준이 있으면 지식 재사용·상호운용·규제 준수에 유리.
- 직접 만들 때: 자연어(저비용·기계 비가독), 또는 RDF Schema/OWL(온톨로지)·SKOS(택소노미) 표준 언어(소프트웨어 지원·자동 추론).

## Essential Characteristics / RDF
> 좋은 지식그래프는 유연하고 유지보수가 쉽고 성능이 빨라야 하며, 사용을 통해 끊임없이 풍부해지는 "살아있는" 자산이다.

- 특정 기술(RDF triplestore)이 필수는 아니다 — 저자들은 도구 지원과 커뮤니티가 풍부한 속성 그래프 모델을 선호.
- RDF(Resource Description Framework)는 subject-predicate-object 트리플 직렬화로, "데이터 교환 포맷"이지 저장/쿼리 모델이 아니다. 와이어 포맷이 데이터 모델을 지배해선 안 된다.

## Summary (핵심 정리)
- 데이터를 똑똑하게 만드는 것은 조직 원리이며, Property Graph → Taxonomy → Ontology로 갈수록 추론력이 커지지만 그만큼 복잡도와 비용도 증가한다.
- 모델링의 황금률은 "just-enough semantics + 반복적 진화" — 처음부터 완벽한 온톨로지를 만들지 말고 유스케이스가 요구할 때 의미를 덧붙여라.
- 주식시장 도메인에서는 금융 표준 온톨로지 FIBO를 출발점으로 채택하고, 산업 분류 택소노미와 공급/경쟁/노출 관계를 점진적으로 얹어 자본시장의 구조적 맥락을 모델링하는 것이 실무적 정석이다.
