# 05. Semantic Model Development

## 챕터 개요 (3줄 요약)
- semantic model 개발은 일회성 engineering이 아니라 지속적 노력이 필요하며, 기술뿐 아니라 business·전략·조직 측면을 함께 고려해야 한다.
- 개발은 6 활동의 반복 — setting the stage, deciding what to build, building it, ensuring it's good, making it useful, making it last.
- upper ontology·design pattern·standard model 등 기존 자원을 재사용하고, IE/NLP/ML 기반 semantic model mining으로 개발을 자동화·확장한다.

---

## 1. Development Activities
> 저자는 모든 개발 project를 6 활동의 iteration으로 접근한다.

- 3대 교훈: ① off-the-shelf 방법론을 문제에 강요 말고 문제 context에 맞게 적응, ② model은 거의 항상 일회성이 아닌 지속적 노력 필요, ③ 거의 항상 순수 기술이 아닌 business·전략·조직 고려 필요.

### 1.1 Setting the Stage
> 개발 전 5가지 핵심 질문(what, why, how, who, who cares)에 명확한 답을 얻어 전략을 정의한다.

- **What**: 자명하지 않음 — "knowledge graph"도 사람마다 정의가 다름(RDF graph, DB 유사 구조, entity network 등). 옳은 정의 논쟁이 아니라 client의 실제 의도를 파악해 stakeholder에게 전달. model만인지 기술·process 포함인지 명확히.
- **Why**: 기술·business 목표 — 요구사항·challenge의 윤곽을 줌(ecommerce 카테고리 taxonomy vs chatbot 도메인 지식은 ambiguity·vagueness 처리 필요).
- **How**: 상세 계획이 아닌 philosophy·원칙. 예(Textkernel): ① graph는 product가 처리할 실제 data·사용 방식이 주도, ② 다양한 구조·비구조 source mining으로 생성·갱신(대규모·volatile 도메인엔 top-down expert 방식 비현실적), ③ automatic mining + human-in-the-loop QA. 기대 정렬·오해 방지에도 유용.
- **Who**: how가 결정 — expert 입력이면 domain expert, unstructured mining이면 NLP/ML, 강한 axiom이면 formal logic 전문가(Ch11).
- **Who cares**: stakeholder analysis — 참여·관심·영향 수준별 분류.

### 1.2 Deciding What to Build
> model의 requirement(원하는 data·답할 질문·도메인·지원 app·품질)를 구체화한다.

- 모호한 요구는 개발 전 tighten("cinema 모델"엔 시기·지리 범위 필요). non-functional requirement(framework, reasoning 복잡성, 정렬할 third-party model)도.
- 준비할 3가지: ① 요구사항은 "수집"이 아닌 "조사·발견"(Ch8), ② 동시 충족 불가한 conflicting requirement 불가피(본서 Dilemmas 파트의 존재 이유) → 조기 해결, ③ 불가능·극난 요구(95% precision synonym 자동 추출) → feasibility·priority 분석 동반.

### 1.3 Building It
> 요구를 만족하는 modeling element를 선택·정의·조합하고, knowledge acquisition 메커니즘을 구현한다.

- 예: film/book character + isEquivalentTo relation, 또는 EN/FR/IT 다국어 lexicalization. source(expert·data·user)에서 element 생성하는 process 설계(DBpedia extractor, Textkernel의 semi-automatic 발견, ESCO의 분산 expert 협업).
- building 중 초기 requirement가 비현실적·충돌임을 발견해 revise하게 됨.

### 1.4 Ensuring It's Good
> 가장 중요한 품질 dimension·metric을 정의하고 측정 메커니즘을 구현한다(Ch4).

- 어느 dimension이 더 중요한지 결정·revise — accuracy와 completeness를 동시 달성 못 해 trade-off 필요(Ch9).

### 1.5 Making It Useful
> 고품질 model이라도 실제 user·system이 쓰고 이득을 주지 않으면 무용하다.

- 초점: app 작동 세부 이해, model-app 간 비호환점 식별·극복(business ontology의 높은 ambiguity로 disambiguation 모듈 신규 개발), app 간 conflicting requirement 조정, model 적용 전후 end-to-end 품질 비교·개선 입증.

### 1.6 Making It Last
> 배포 후에도 지속 개선·change-management로 model의 longevity를 보장한다.

- 3 이유: ① 첫 버전은 완벽하지 않아 feedback·개선 체계 필요, ② 도메인 변화에 timely 유지(semantic drift, Ch4), ③ requirement 변화(도메인 확장, 언어 변경, 표준 매핑). evolution dilemma·전략은 Ch14.

---

## 2. Vocabularies, Patterns, and Exemplary Models
> 바퀴를 재발명하지 말고 기존 semantic 자원을 재사용하면 개발을 가속하고 interoperability를 높인다(Ch8).

- **Upper ontologies**(top-level/foundational): 도메인 무관 일반 concept(DOLCE, BFO). 여러 model이 같은 upper ontology에 연결되면 interoperability 향상. 단 추상적·인식론적이라 개발·선택·연결 비용 큼, 표준 부재.
- **Design patterns**: 반복 문제의 재사용 가능 검증된 해법. 예: OWL의 ternary relation 미지원 → n-ary relation pattern(married를 Wedding class로, 특정 결혼을 instance로). 분류 기준: 대상 문제 종류(언어 미지원 보완 vs good-practice vs 도메인 문제), 도메인·적용 범위, 표현 언어(Silverston/Hay는 E-R, W3C/Manchester/OntologyDesignPatterns.org는 OWL). 정확히 맞는 pattern 찾기는 어려울 수 있음.
- **Standard·reference models**: 도메인·산업의 합의 model(SNOMED CT, Schema.org, ISCO, FIBO, HL7 RIM).
- **Public models·datasets**: Linked Open Data(RDF/OWL, open license — DBpedia, GeoNames, Diseasome, CrunchBase, MusicBrainz), Linked Open Vocabularies(주로 class, content보다 structure 지원 — Music Ontology).

---

## 3. Semantic Model Mining
> 대규모 model은 human-only가 너무 비싸므로(Cyc 21M statement에 $120M vs DBpedia 400M을 더 적은 비용에 Wikipedia 추출), IE/NLP/ML로 제한된 human 노력으로 element를 mining한다.

### 3.1 Mining Tasks
> Ch2의 element에 대응하는 4대 IE task가 개발을 자동화한다.

- **Terminology extraction**(도메인 관련 term 추출, bottom-up), **Entity extraction**(=entity recognition, 알려진 type으로 — skill 발견), **Relation extraction**(synonymy·meaning inclusion·relatedness 등), **Rule and axiom extraction**(reasoning용 복잡 axiom).
- 예(Waterloo 텍스트): terminology가 Napoleon·Prussian Army·command 등, entity가 Napoleon=Person·Waterloo=Location, relation이 "Waterloo located in Belgium".
- **mention-level vs global-level**: model mining엔 보통 global-level(어디서 찾았는지 무관한 distinct element 목록) 필요.
- 효과 좌우 요인: target 정보 복잡성(term<type, binary<event), 특정성(similar<synonym), input data 적절성(synonym은 encyclopedic이 news보다 쉬움), 구조 정도(infobox가 자유 텍스트보다 쉬움). task 순서는 정해진 게 없어 pipeline orchestration이 challenge.

### 3.2 Mining Methods and Techniques
> 모든 IE system은 extraction pattern을 가지며, 그것을 어떻게 획득하느냐가 다르다.

- **Hand-built patterns/rules**: 수작업 규칙. **Hearst patterns**(is-a — "X and other Y", "Y such as X"), part-whole·cause-effect·location pattern. 단순·빠름·well-defined 도메인에 고정확하나, 전이성 낮고 coverage 낮고 수작업 많음.
- **Supervised ML**: human-labeled data로 추출 규칙 학습(Stanford NER 7 type, MUC dataset). 복잡 pattern 학습·도메인 적응 가능하나 training data 비쌈·도메인 bias.
- **Semi-supervised(weakly supervised)**: 소수 seed로 초기 pattern→더 많은 labeled data→새 pattern 반복(NELL — 800 class에서 시작, 현재 2.8M instance). 수작업 절감·unlabeled data 활용하나 **error propagation** 위험(seed 선택 중요, NELL은 75% class에 고precision·25%에 저precision).
- **Distant supervision**: 기존 model로 labeled data 자동 생성(Mintz et al. — Freebase 300 relation으로 sentence 추출해 classifier 학습). 수작업 불필요·대규모 적용하나, 같은 entity쌍의 다중 관계·ambiguity·기존 model 오류가 training data 오염, 고정 relation 집합이라 신규 도메인 적응 어려움.
- **Unsupervised**: labeled data·규칙 불필요, 통계적 semantics·unsupervised ML로 pattern 발견(Hasegawa의 context clustering, **Word2Vec** word embedding — 공통 context 공유 term이 가까운 vector). annotation 불필요·대규모 쉬우나, 추출 정보가 incoherent·disambiguate 어려움(Biology 관련어에 synonym·broader·narrower 혼재).
- **Open Information Extraction(OIE)**: 소수 well-defined target·동질 corpus가 아닌 대규모 이질 corpus의 모든 element 추출 철학(TextRunner, StatSnowball, ReVerb — 대개 unsupervised).

---

## Summary (핵심 정리)
- semantic modeling project는 일회성·순수 기술이 아니며, 문제 context에 방법론을 맞춰야 한다.
- 개발은 setting the stage→deciding→building→ensuring quality→making useful→making it last의 6 활동 반복이다.
- what/why/how에 구체적 답을 강제해 오해·기대를 관리한다.
- upper ontology·design pattern·standard·LOD 등 기존 자원을 재사용하되 맹신하지 않는다.
- terminology/entity/relation/axiom extraction을 hand-built·supervised·semi-supervised·distant·unsupervised 방법으로 mining해 개발을 자동화·확장한다.
