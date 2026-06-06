# 07. Named Entity Disambiguation

## 챕터 개요 (3줄 요약)

- NER(Named Entity Recognition)은 엔티티를 인식하나 의미 모호성을 해소하지 못하므로, 문맥을 보고 지식 베이스의 정확한 엔티티에 연결하는 NED(Named Entity Disambiguation)가 필요하다.
- 유럽 SoHO(Substances of Human Origin) 규제 도메인을 사례로, scispaCy로 의료 엔티티를 해소하고 UMLS·SNOMED·HPO 온톨로지를 통합해 비정형+구조화 지식의 통합 KG를 구축한다.
- 완성된 KG로 개념 검색, 구조화 지식 검색, 해석가능성·발견, 새로운 지식 발굴이라는 네 가지 고급 활용 사례를 수행한다.

---

## 1~2. From Recognition to Disambiguation / Understanding NED

> NED는 각 언급의 문맥을 검토해 의미 불확실성을 제거하고 지식 베이스의 엔티티에 연결한다.

- "Zika"가 한 텍스트에 세 번 등장하나 문맥에 따라 바이러스/질병/선천성 형태로 의미가 달라지며, NER만으로는 구분 못 한다.
- NED 매핑으로 IAS의 두 요구(의미 있는 엔티티 탐지, 다양한 지식 소스에서 정보 검색)를 충족하고, UMLS 엔티티에서 생의학 온톨로지의 맥락 지식을 탐색한다.
- 반대로 다른 용어가 같은 엔티티를 가리킬 수도 있다(AIDS = Acquired Immunodeficiency Syndrome).
- NED는 세 단계: 후보 선택(candidate selection), 후보 순위화(candidate ranking, 문맥 점수), 온톨로지 통합(ontology integration).
- scispaCy(en_core_sci_md)로 UMLS 대상 후보 선택·순위화를 수행하며, 각 "Zika" 언급에 다른 UMLS CUI(Concept Unique Identifier)를 할당한다.
- UMLS는 여러 어휘를 매핑하는 메타시소러스이며, SNOMED(45만+ 개념, CAUSATIVE_AGENT·FINDING_SITE 등 관계)는 가장 포괄적인 임상 용어 체계다.

---

## 3. Domain-based NED and LLMs

> LLM은 scispaCy의 현대적 대안이나 가치를 더하려면 KG 기술과 결합해야 한다.

- ChatGPT는 첫 "Zika"가 바이러스, 마지막이 감염임을 완벽히 구분하지 못한다.
- ChatGPT는 UMLS ID를 할당하지 못하며, 도메인 특화 지식·전문성이 필요하다고 스스로 답한다.
- 따라서 NED 기술과 KG를 결합한 접근이 이런 특징을 갖추고 IAS에 통합되기 쉽다.

---

## 4. Business and Domain Understanding

> 실제 시나리오는 SoHO(혈액·조직·세포·장기) 관리의 표준·규제 정의로, 수혈·이식·보조생식의 환자 안전과 직결된다.

- BTC(Blood, Tissues, Cells) 부문은 시민 기부에 의존하나 COVID-19 같은 위기 시 가용성이 줄어, 미래 대비·위기 저항적 법적 틀이 필요하다.
- 2022년 EC(European Commission)가 SoHO 품질·표준 규제안을 발표했고, ECDC(질병 감시 보고)와 EDQM(품질·안전 가이드라인)이 상호 보완 역할을 한다.
- 활용 사례: 보건 정책 담당자가 췌도(islets of Langerhans) 이식 가이드라인·위험과 특정 지역 Zika 확산을 분석.
- 개념 검색(의미 기반), 구조화 지식 검색(온톨로지 형식 지식), KG 기반 해석가능성·발견, 새로운 지식 발굴이 핵심 기능이다.

---

## 5. Understanding the Data

> SoHO IAS는 여러 저장소의 이질적 비정형·구조화 데이터를 통합해야 한다.

- 비정형 데이터: BTC 영향평가 보고서·규제안, 이해관계자 입장문, EDQM 가이드라인·뉴스레터, ECDC 보고서·CDTR(주간 전염병 위협 보고).
- 도메인 온톨로지: UMLS(MRCONSO.RRF 엔티티 목록, MRSTY.RRF 의미 타입), SNOMED(Description·Relationship 파일), HPO(hp.owl).
- 각 엔티티는 UMLS ID로 식별되며 SNOMED·FMA·MSH·HPO 등 다양한 소스 코드에 매핑된다.

---

## 6. Building a SoHO Knowledge Graph

> KG 구축은 스키마 정의→문서 처리·수집→의료 엔티티 해소·수집→온톨로지 처리·로딩·매핑→동시출현 관계 생성 순으로 진행된다.

- 스키마: File→Page→EntityMention→(DISAMBIGUATED_TO)→MedicalEntity, MedicalEntity는 IS_SNOMED_ENTITY·IS_HPO_ENTITY·IS_DISEASE_ENTITY로 온톨로지에 매핑된다.
- 문서는 Amazon Textract OCR로 텍스트 추출 후 1열/2열 구조를 재구성해 Neo4j에 적재한다.
- scispaCy 처리 결과(언급 위치·UMLS ID·정의·별칭·의미 타입)를 EntityMention·MedicalEntity 노드로 적재한다.
- SNOMED는 단일 SNOMED_RELATION(type 속성)으로 단순화하고, SNOMED_IS_A로 계층을 추적하여 루트(질병·신체구조 등)의 의미 타입을 하위 노드로 전파한다.
- HPO는 Neosemantics로 import 후 표현형 특징·질병 노드와 HAS_PHENOTYPIC_FEATURE 관계를 만든다.
- 동시출현(co-occurrence)은 같은 문장 내 의료 엔티티를 COOCCURR 관계(count·sentences)로 연결하며 2.5만+ 관계를 생성한다.

```
  File -> Page -> EntityMention --DISAMBIGUATED_TO--> MedicalEntity
                                                          |
   IS_SNOMED_ENTITY / IS_HPO_ENTITY / IS_DISEASE_ENTITY   |
                                                          v
                                  SnomedEntity / HpoEntity / HpoDiseaseEntity
  MedicalEntity --COOCCURR(count)-- MedicalEntity  (same sentence)
```

---

## 7. KG-based Use Cases

> NED와 KG를 결합해 네 가지 활용 사례를 수행한다.

### 7.7.1 Conceptual Search

- 전통 전문검색(full-text)은 "breakbone fever"에서 'fever'만 매칭하나, 개념 검색은 UMLS 별칭(C0011311=dengue fever)으로 정확한 엔티티·언급 위치·횟수를 검색한다.
- "islands"가 "islands of Langerhans"에서 다른 의미를 갖는 경우도 개념 검색이 무관한 결과를 거른다.

### 7.7.2 Structured Knowledge-Based Search

- 온톨로지 형식 지식으로 여러 문서의 텍스트를 비자명하게 연결한다(예: islets of Langerhans에 영향 주는 질병=FINDING_SITE 관계로 당뇨·고혈당 검색).
- 온톨로지의 긴 경로(CAUSATIVE_AGENT 연쇄)를 활용해 Togavirus가 일으키는 모든 질병(Yellow fever, Rubella 등) 언급 문서를 검색한다.

### 7.7.3 KG-based Interpretability and Discovery

- 동시출현 엔티티를 온톨로지로 분석: 해석가능성(co-occurrence 이유 설명, 예: AIDS·Hepatitis가 모두 감염병)과 발견(텍스트를 넘어선 지식 확장)을 제공한다.
- Zika virus disease와 Zika virus는 CAUSATIVE_AGENT로 직접 연결되어 동시출현을 "해석"하며, Dengue-Zika는 Flavivirus·Togavirus·Mosquito-borne 등 여러 경로로 연결된다.
- HPO 동시출현(표현형 특징-질병)도 체계적으로 추출·검증할 수 있다(예: Renal cell carcinoma-von Hippel-Lindau syndrome).

### 7.7.4 Uncovering New Knowledge

- 연구 발전으로 온톨로지에 아직 굳어지지 않은 지식은 동시출현 엔티티가 직접 매치되지 않을 수 있다.
- Guillain-Barre Syndrome은 매개체 전파 질병이 아닌데도 Zika와 자주 동시출현하여, 합병증 가능성을 시사한다.
- 텍스트는 Zika가 Guillain-Barre의 원인 중 하나라 하나 SNOMED엔 직접 CAUSATIVE_AGENT 연결이 없으며, GDS degree 알고리즘으로 허브 노드(Inflammation 등)를 걸러 11,185개에서 9개 경로로 줄여도 직접 연결은 없다.
- 이는 KG의 동시출현이 도메인 온톨로지를 풍부하게 하는 선순환(virtuous circle)의 전형적 예다.

---

## Summary (핵심 정리)

- NED는 텍스트의 엔티티를 참조 지식 베이스에 연결하며, KG 기술과 결합하면 중요 도메인에서 고급 서비스 개발 기회를 연다.
- KG 구축은 스키마 정의·문서 수집·엔티티 해소·도메인 온톨로지 통합·매핑·동시출현 관계 생성의 여러 단계를 요한다.
- 완성된 KG로 개념 검색, 구조화 지식 검색, 해석가능성·발견, 새로운 지식 발굴이라는 네 활용 사례를 수행할 수 있다.
