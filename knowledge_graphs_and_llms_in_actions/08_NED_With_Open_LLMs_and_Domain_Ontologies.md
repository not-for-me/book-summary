# 08. NED with Open LLMs and Domain Ontologies

## 챕터 개요 (3줄 요약)

- scispaCy 같은 전통 NED 도구는 생의학에 국한되고 지식 베이스 확장이 어려우며 엔티티 간 관계·경로를 활용하지 못하는 한계가 있다.
- 오픈 범용 LLM(Llama 3.1 8B via Ollama)과 도메인 온톨로지(SNOMED)를 결합해 다양한 도메인에 적용 가능한 NED 시스템을 구축한다.
- 후보 명확화(candidate disambiguation)는 최단 경로 탐지→경로의 텍스트 변환→텍스트 요약이라는 다단계로 그래프와 LLM 강점을 결합한다.

---

## 1. Understanding Limitations of Traditional NED Systems

> scispaCy는 UMLS 등 어휘·온톨로지를 통합하나 여러 한계를 갖는다.

- 특정 도메인(생의학)에 설계되었고, 참조 지식 베이스의 새 엔티티·용어 확장·갱신이 어렵다.
- 지식 베이스의 방대한 정보를 충분히 활용하지 못하고, 엔티티 간 관계·경로를 명확화에 쓰지 못한다.
- "congenital"·"syndrome" 같은 주변 단어가 없으면 scispaCy는 "Zika disease" 언급의 타깃 엔티티를 탐지하지 못한다.
- 본 장은 오픈 LLM과 도메인 온톨로지로 이 한계를 해결하며, 풍부한 온톨로지가 있는 다른 도메인에도 적용 가능하다.

---

## 2. Ingesting the Domain Ontology

> 명확화를 이끌기 위해 SNOMED(45만+ 개념, 풍부한 관계 타입) 온톨로지를 사용한다.

- 7장과 동일하게 Relationship·Description 파일을 적재하며, 단일 SNOMED_RELATION과 SNOMED_IS_A 관계를 생성한다.
- SNOMED_IS_A 관계로 루트 노드(질병·신체구조 등 원형 엔티티)의 의미 타입을 계층 구조를 통해 하위 노드로 전파한다.

---

## 3. Setting up the Model with Ollama and Llama 3.1 8B

> Ollama는 LLM을 로컬에서 실행하는 오픈소스 도구로, 데이터 통제·낮은 지연·외부 의존 감소를 제공한다.

- Llama 3.1 8B는 80억 파라미터 오픈소스 LLM으로 최대 12.8만 토큰 컨텍스트와 다국어 처리, 소비자급 하드웨어 배포에 최적화되어 있다.
- Ollama는 OpenAI Chat Completions API 호환이라 이전 장의 파이썬 코드로 로컬 모델과 직접 상호작용한다(temperature=0).
- 범용 모델 Llama 3.1을 NED에 써서 도메인 온톨로지와 결합 시 틈새 영역에서도 LLM이 잘 작동함을 보인다.

---

## 4. End-to-End NED Process

> 입력 문서→LLM 기반 NER→후보 선택(CS)→후보 명확화(CD)로 진행되며 각 단계에 도메인 온톨로지를 통합한다.

```
  Input Text --> NER (LLM + ontology categories)
              --> Candidate Selection (Neo4j full-text search)
              --> Candidate Disambiguation
                    (1) shortest paths (GDS)
                    (2) path-to-text (LLM)
                    (3) summarize text (LLM)
                    (4) final disambiguation (LLM)
              --> Disambiguated SNOMED entities
```

### 8.4.1 Named Entity Recognition (NER)

- NER은 비정형 텍스트의 명명 엔티티를 질병·유기체·시술 등 사전 정의 범주로 식별·분류한다.
- SNOMED의 1단계 노드에서 범주를 추출(propagation 결과)하여 프롬프트에 명시하고, 문장 단위로 JSON(sentence·entities[id·mention·label])을 출력한다.
- LLM은 언급의 시작·끝 문자 위치를 정확히 못 잡으므로, 파이썬 함수로 후처리하여 start·end를 계산한다.

### 8.4.2 Candidate Selection (CS)

- 각 언급에 대해 의도된 의미와 일치할 후보 엔티티·개념을 식별한다.
- LLM을 쓰지 않는 이유: 온톨로지에서 직접 후보를 검색하고자 하며, 온톨로지가 커서 프롬프트에 전부 로드할 수 없기 때문이다.
- Neo4j 전문 검색(full-text search)으로 언급과 유사한 문자열을 찾고, NER의 label로 검색 공간을 줄인다(벡터 검색으로 강화 가능).
- "Zika" 입력 시 Zika virus(50471002)·Zika virus disease(3928002)·Congenital Zika virus infection(762725007)을 후보로 반환한다.

### 8.4.3 Candidate Disambiguation (CD)

- 같은 문장에 동시출현하는 다른 의료 엔티티의 맥락 정보를 활용해 후보를 검증·정제한다(예: "microcephaly"가 함께 있으면 Congenital Zika virus infection 우선).
- 최단 경로 탐지: GDS degree 알고리즘으로 허브 노드(상위 350개)를 제외하고, 후보 간 1~2홉 최단 경로를 SNOMED_RELATION으로 찾아 가독 문자열로 변환한다.
- 경로→텍스트 변환: 그래프 경로를 LLM이 잘 처리하는 자연어 문장으로 변환해 관계 정보를 해석 가능하게 한다.
- 텍스트 경로 요약: 변환된 문장들을 짧은 요약으로 합쳐 모델의 "인지 부하"(토큰 수)를 줄인다.
- 최종 명확화: 원문장·후보 엔티티·맥락 문장(요약)을 결합해 LLM이 각 언급에 가장 적합한 SNOMED 엔티티를 선택한다(예: "Zika"→Congenital Zika virus infection).

---

## 5. Conclusions

> SNOMED 같은 도메인 온톨로지와 Llama 3.1 8B 같은 오픈 범용 LLM을 통합해 scispaCy 등 전통 NLP 도구의 한계를 해결한다.

- Neo4j GDS의 최단 경로 탐지·전문 검색과 LLM의 명확화 능력을 결합해 복잡한 텍스트의 엔티티를 견고하게 식별·명확화한다.
- 경로→텍스트 변환·텍스트 경로 요약 기법으로 LLM이 관계형 데이터를 자연어 형식으로 처리하는 능력을 향상시킨다.
- 이 프레임워크는 풍부한 온톨로지가 있는 다른 도메인의 NED 작업에 적용 가능한 토대를 제공한다.

---

## Summary (핵심 정리)

- NED는 복잡한 도메인에서 엔티티를 정확히 식별·구별하는 데 필수이며, scispaCy 같은 전통 도구는 도메인 제약·관계 미활용·지식 갱신 불가의 한계가 있다.
- 범용 LLM과 도메인 온톨로지의 결합은 지속 갱신되는 온톨로지 지식과 관계 구조를 LLM에 제공하여 이 문제들을 해결한다.
- 명확화는 최단 경로 탐지·경로의 텍스트 변환·텍스트 경로 요약 3단계로 나뉘며, 풍부한 온톨로지를 가진 다른 도메인에도 적용·확장할 수 있다.
