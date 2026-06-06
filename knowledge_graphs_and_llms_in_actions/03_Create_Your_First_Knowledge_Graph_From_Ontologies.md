# 03. Create Your First Knowledge Graph from Ontologies

## 챕터 개요 (3줄 요약)

- 이질적 데이터의 의미 차이는 온톨로지를 참조 스키마·어휘로 채택하는 의미적 통합(semantic integration)으로 해결하며, 본 장은 희귀질환 진단 지원 KG를 구축한다.
- HPO(Human Phenotype Ontology)와 주석 데이터를 수집·처리하며, RDF와 LPG(Labeled Property Graph) 기술을 목표 주도로 비교해 엣지 속성 표현에 유리한 LPG/Cypher를 선택한다.
- 구축된 KG에서 질의와 온톨로지 기반 추론(inference)을 수행하여, 명시되지 않은 함축적 연결까지 끌어내 임상의의 진단을 지원한다.

---

## 1. Knowledge Graph Building: Warmup

> KG 구축 전 해결할 문제를 분석하고 응용 도메인 개요를 만들며 데이터를 탐색한다.

- 대상 페르소나는 임상의(clinician)이며, 핵심 과제는 증상(표현형 특질, phenotypic trait)으로 질병, 특히 희귀 증후군을 정확히 식별하는 것이다.
- 필요한 지식 베이스는 표현형 도메인의 맥락적 기술(같은 장기 관련 이상은 명시적 연결)과 표현형 이상-질병 관계 데이터(출처 추적 가능)를 갖춰야 한다.
- 표현형(phenotype)은 개체가 나타내는 모든 표현형 특징의 합이며, 질병(disease)은 원인·시간 경과·표현형 특징군·특정 치료 반응으로 특징지어지는 엔티티다.
- 회색 영역 존재: 당뇨병처럼 질병이자 다른 희귀 증후군의 표현형 특징이 될 수 있어 맥락에 따라 두 개의 다른 ID로 표현된다.
- 데이터 통합 장애물: 같은 개념의 다양한 표현(type 2 diabetes vs ketosis-resistant diabetes), 동일 약어의 다른 개념(PE=신체검사 또는 폐색전증), 정보 입도(granularity) 차이.

### 3.1.2 Data Understanding

- 데이터 소스는 HPO 저장소로, RDF/XML 파일 hpo.owl(표현형 이상의 표준 온톨로지)과 TSV 파일 phenotype.hpoa(질병별 주석된 표현형 특징)를 제공한다.
- OWL 파일은 rdflib 파이썬 라이브러리로 주어-술어-목적어(subject-predicate-object) 트리플 집합으로 탐색할 수 있다.
- HPOA 파일은 database_id(OMIM), disease_name, hpo_id, reference(PMID), evidence(PCS=published clinical study), frequency, aspect, biocuration 필드를 포함한다.

---

## 2. Understanding Knowledge Graph Technologies

> KG 생성의 두 주요 기술 RDF(triple 기반, W3C 표준)와 LPG(key-value 속성 기반)를 이해한 뒤 용도에 맞게 선택한다.

- RDF는 주어-술어-목적어 트리플로 KG를 진술의 집합으로 모델링하며 웹 기술로 표현·저장·교환하고 온톨로지 기술에 적합하다.
- LPG는 노드·관계에 key-value 쌍을 부여해 빠른 질의 기반 순회(traversal)와 경로 분석을 보장한다.
- RDF는 관계(트리플)가 전역 정의되어 술어 메타데이터가 모든 인스턴스에 영향을 주며, 이를 완화하기 위해 named graph를 지원한다.
- LPG는 노드 간 고유 엣지를 지원해 개별 관계에 메타데이터·속성을 붙일 수 있다.
- RDF-star는 엣지에 속성을 추가해 RDF와 LPG 간극을 좁히는 확장이며, Neo4j의 Neosemantics 플러그인은 RDF 어휘(OWL, RDFS, SKOS)로 기본 추론을 가능케 한다.

### 3.2.1 ~ 3.2.2 RDF or LPG? Edge Properties

- 본 사례 요구사항: 임상의는 표현형 특징(또는 조합)이 탐지 어려운 질병과 연관된 사례를 출처·날짜와 함께 보고 비교하기를 원한다.
- 주석은 특정 표현형 특징과 질병의 연결을 출처(provenance)·날짜와 함께 표현해야 하므로, 이를 디스이즈-표현형 간 관계에 통합하는 것이 최적이다.
- RDF로 엣지 정보를 표현하는 방법: n-ary 관계(annotation 개념 신설), named graph(네 번째 요소로 하위 그래프 지정), RDF-star(<<...>> 구문으로 트리플에 속성 부여).
- RDF-star는 가독성은 좋으나 질의 성능 개선이 필요하고 새 구문 확장이 엔진별 구현을 요구해 채택이 제한된다.
- LPG는 관계 내부에 key-value로 주석 세부정보를 직접 표현해 메타데이터가 풍부한 관계 모델링에 가장 적합하므로, 본서는 LPG와 Cypher를 핵심 도구로 채택한다.

```
  Table Row (annotation)            KG Edge (LPG)
  +-------------------+      (Disease)
  | disease=OMIM:222100|  --[:HAS_PHENOTYPIC_FEATURE
  | hpo=HP:0410050     |       {source, createdBy, creationDate}]-->
  | source=PMID:9357814|     (Phenotype)
  +-------------------+
```

---

## 3. Building a Knowledge Graph

> KG 구축은 온톨로지 로딩과 온톨로지를 참조로 한 데이터 소스 수집의 두 단계로 진행된다.

- 환경: Neo4j(5.20.0 Enterprise) + APOC 라이브러리 + Neosemantics 플러그인을 사용하며, Cypher 질의로 구축한다.
- 온톨로지 수집: hpo 데이터베이스 생성, Resource 노드의 uri/id 유일성 제약과 인덱스 생성, Neosemantics 설정(namespace 무시, 관계 타입 대문자화) 후 hp.owl을 import한다(약 899,558개 진술).
- 노드 강화: HP로 시작하는 URI 노드에 HpoPhenotype 레이블과 id 속성(예: HP:0000001)을 부여한다.
- 주석 수집: TSV 파일에서 HpoDisease 노드 생성, HpoDisease-HpoPhenotype 간 HAS_PHENOTYPIC_FEATURE 관계 생성, null이 아닌 컬럼만 속성으로 설정하는 FOREACH 패턴으로 견고하게 처리한다.
- apoc.periodic.iterate로 biocuration에서 정규식으로 createdBy/creationDate를 추출하고, aspect(P/I)·evidence(IEA/PCS/TAS) 약어를 사람이 읽을 수 있는 이름·설명으로 확장한다.
- 마지막으로 HpoPhenotype·HpoDisease가 아닌 불필요한 온톨로지 노드·관계를 DETACH DELETE로 정리한다.

---

## 4. Querying the Data

> 임상의는 환자의 표현형 이상을 입력해 KG에 질의함으로써 희귀 질환을 식별할 수 있다.

- 병원이 KG 패러다임을 채택해 환자 정보를 HPO·OMIM 용어로 저장하며, Type 1 당뇨는 표현형 특징(HP:0100651)과 질병(OMIM:222100) 두 코드로 저장된다.
- 첫 질의는 특정 질병(OMIM:222100)에 연결된 모든 표현형 특징을 검색한다.
- 임상의가 추가 증상(성장 지연, 큰 무릎, 감각신경성 난청, 가려움증)을 발견하면, 이 표현형 특징들과 연관된 질병을 일치 개수 순으로 검색한다.
- 결과 1위는 5개 특징이 모두 일치한 'Ondontochondrodysplasia 2 with hearing loss and diabetes'(OMIM:619269)로, 이를 출발점으로 추가 조사가 가능하다.

---

## 5. Reasoning over the KG

> KG의 가장 강력한 도구 중 하나는 논리 규칙 기반 연역 추론으로 함축적 정보에서 결과를 도출하는 추론(inference)이다.

- "내분비계 이상을 특징으로 하는 질병은?" 같은 질문에서, 명시적 연결뿐 아니라 갑상선 등 더 구체적인 하위 표현형 특질도 관심 대상이다.
- HPO의 계층 구조(SUBCLASSOF)를 활용해 내분비계 이상(HP:0000818)의 1~3단계 하위 클래스 표현형 특징을 검색한다.
- Neosemantics의 n10s.inference.nodesInCategory 프로시저로 inCatRel(HAS_PHENOTYPIC_FEATURE)과 subCatRel(SUBCLASSOF)을 지정해, 직접·간접적으로 연결된 질병을 추론한다.
- 이를 통해 직접 연결을 넘어 도메인 지식 구조를 활용한 의미적 추론(semantic inference)으로 의미 있는 질병 연관을 발견한다.

---

## Summary (핵심 정리)

- KG 구축은 해결할 문제·참조 도메인 이해와 데이터 탐색·이해 단계를 요하며, 결과는 여러 소스의 정보가 단일 뷰로 융합된 통일적·근거 있는 표현이어야 한다.
- RDF는 지식 표현·온톨로지 구축에, LPG는 빠른 질의 기반 순회·경로 분석에 적합하며, 둘의 차이 이해가 목적별 기술 선택에 결정적이다.
- 메타데이터가 풍부한 관계 표현이 핵심인 본 사례에서는 LPG/Cypher가 선택되며, 온톨로지의 계층 구조를 활용한 의미적 추론이 함축적 질병 연관을 드러낸다.
