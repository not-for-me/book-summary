# 06. Building Knowledge Graphs with Large Language Models

## 챕터 개요 (3줄 요약)

- 5장에서 추출한 RAC(Rockefeller Archive Center) 지식을 정규화·정제·엔티티 해소를 거쳐 3계층 KG로 변환하고 분석한다.
- 그래프 기반 엔티티 해소(graph-based entity resolution)는 문자열 유사도와 관계(WORKS_FOR, WORKS_ON) 신호, 커뮤니티 탐지, 임베딩을 결합해 동일 인물을 식별한다.
- 완성된 지적 영향 네트워크(intellectual network)를 그래프 데이터 과학(PageRank, 매개 중심성 등)으로 분석하며, LLM 시대에도 KG가 설명가능성·민주화·고급 분석의 가치를 제공함을 보인다.

---

## 1. Transforming an Archive to a KG

> LLM은 원하는 출력을 최대한 생성하나 KG를 직접 만들 수는 없으며, 정규화·엔티티 해소 단계가 선행되어야 한다.

- 남은 과제: 아날로그 타자 문서 OCR 처리, 더 이상 추구되지 않는 역사적 연구 분야(참조 지식 베이스 부재), 비정상적 표기 관습(약어), 도메인 특화 엔티티(Occupation), 높은 관계 복잡성, KG 정규화·정제·해소, 비정형 소스 매칭·연결.
- LLM 시대 덕분에 과거 전통 ML 모델로 큰 자원을 들이던 많은 장애물을 지식 표현 시스템·LLM·프롬프트 엔지니어링의 올바른 선택으로 해결할 수 있다.

### 6.1.1 Graph Modeling

- KG를 3계층으로 설계: 문서 계층(File·Page 노드), 메타그래프 계층(병합 안 된 GPT Entity 노드·관계와 원본 페이지 연결), KG 계층(최종 정규화·해소된 Person·Title·Organization·Occupation과 관계).

```
  Document Layer:  File --> Page (OCR text)
       |
  Metagraph Layer: Entity --RELATED_TO_ENTITY--> Entity (linked to Page)
       | (normalize, resolve)
  KG Layer:        Person --WORKS_FOR--> Organization
                          --WORKS_ON--> Occupation
```

### 6.1.2 Creating a Metagraph

- 긴 텍스트는 max token 한계·품질 저하를 피하려 청킹(chunking) 전략이 필요하며, 가장 단순한 방법은 페이지 단위 분할이다.
- 식별된 엔티티 언급을 병합(merge)이 아니라 생성(create)하여 페이지에 연결하고 관계를 만든다.
- 이를 통해 엔티티·관계의 출처를 추적할 수 있어, 원본 텍스트 스니펫을 보여주는 설명가능한 시각화와 메타그래프로부터 KG 재생성이 가능하다.

### 6.1.3 Normalization and Cleansing

- 메타그래프 통계(클래스별 상위 엔티티·관계)로 구조를 파악하고, 연결성 향상 기회를 구현한다(예: Occupation 소문자화).
- GPT가 분리하라는 지시에도 직함을 이름에 포함시키는 경우가 있어, 직함 토큰(dr., prof. 등)을 제거하는 정제 전략으로 동일인이 여러 노드가 되는 것을 막는다.
- 정규화된 name_normalized 속성을 새로 만들어 KG 계층 연결 시 사용한다.

### 6.1.4 Graph-based Entity Resolution

- LLM은 전체·깨끗한 이름을 반환해 coreference 해소가 암묵적으로 처리되지만, 문서 간 해소("Eleanor Smith" = "E. Smith"?)는 여전히 필요하다.
- 문자열 유사도가 높고 같은 대학에서 일하면(WORKS_FOR) 동일인일 강한 신호이며, 여러 신호를 모으면 해소 신뢰도가 높아진다.
- 성+이름(또는 약어) 조합으로 규칙을 정의해 META_SIMILAR 엣지를 만들되, Foundation 같은 일반 단어는 불용어(stopword)로 처리한다.
- Ernest Lawrence 사례: 세 표기가 대학·cyclotron 연구 연결을 통해 다중 홉(최대 6홉)으로 이어지며, 관계형 DB에서는 어려운 순회다.
- 더 그래프다운 방법: TALKED_ABOUT·TALKED_WITH로 Louvain 커뮤니티 탐지, Occupation 의미 유사도는 GPT 임베딩과 응집 군집화(agglomerative clustering)로 해소한다.
- 기준선: META_PERSONS_SIMILAR 관계 생성→WCC로 동일 KG 엔티티 그룹 식별→그룹 대표 이름(가장 긴 이름) 선택→최종 KG 계층 생성.

---

## 2. Intellectual Network Analysis: The Value of Graphs

> KG의 지적 네트워크는 TALKED_ABOUT·TALKED_WITH·WORKS_WITH·STUDENT_OF 관계로 형성되어 그래프 분석에 적합하다.

- Neo4j GDS로 PageRank, 고유벡터 중심성, 노드 차수, 매개 중심성(betweenness centrality)을 계산해 영향력자(influencer)·피영향자(influencee)·다리(bridge)를 식별한다.
- 매개 중심성 기반 스타일링은 서로 다른 연구자 그룹을 잇는 다리 역할 인물(Niels Bohr, Ernest Lawrence 등)을 강조한다.
- cyclotron 연구 영향 네트워크는 2홉 질의로 탐색하며 Harlow Shapley, James B. Conant 등 인물을 드러낸다.
- 실패 사례: Laurence Irving이 잘못 등장한 것은 RE 오류로, LLM도 실수하므로 분석가가 그래프 내용을 검증하는 피드백 루프 설계가 중요하다.
- 활용 예: 신규 담당자가 Johns Hopkins와 Harvard에 걸친 물리 연구를 위해 3홉 이내 연결자를 찾고, TALKED_ABOUT의 감정 속성으로 균형 잡힌 통찰을 얻는다.

---

## 3. Next Steps in the RAC Project

> 150페이지 결과만으로도 KG의 복잡성을 보여주며, 프로덕션 품질에는 추가 작업이 필요하다.

- 지식 추출 개선(프롬프트 반복·파인튜닝), 다중 페이지 문서 처리(일기 항목 경계 식별), 엔티티 해소 확장(WikiData 대조 disambiguation).
- 이사회 회의록에서 보조금(grant) 정보를 마이닝해 대화와 연결, Occupation 엔티티 해소(임베딩+응집 군집화), Conversation 노드 생성과 후속 체인·보조금 연결.

---

## 4. The Value of Knowledge Graphs in the LLM Era

> 초강력 LLM 시대에도 데이터를 직접 모델에 먹이는 대신 KG를 구축하는 이유가 있다.

- 설명가능성(explainability): 잘 설계된 KG 기반 앱은 근거 데이터·추론을 검사·검증할 수 있고 충돌 소스를 다루며 "사고의 사슬"을 제공한다.
- 탈블랙박스화(demystification): LLM을 파인튜닝해 직접 질문하면 응답 신뢰도를 평가할 수 없으나, 문서에서 사실을 추출해 KG로 만들면 통찰에 확신이 생긴다.
- 민주화(democratization): 비싼 LLM을 KG 생성에 한 번만 쓰고 이후 저렴한 배치 업데이트로 오래 활용해 자금이 적은 조직도 혜택을 본다.
- 탐색가능성(explorability)과 고급 분석(advanced analytics): 그래프는 전역 뷰와 드릴다운을 제공하고, 데이터 과학자가 답 생성을 완전히 통제하며 그래프 기반 분석·ML을 수행하게 한다.

---

## Summary (핵심 정리)

- KG 스키마 설계(그래프 모델링)는 정보를 용도에 최적으로 저장해 KG 생성·정제·정규화·해소를 단순화하고 효율적 다운스트림 분석을 보장한다.
- 텍스트에서 고품질 엔티티 관계를 추출하면 비지도 그래프 기반 엔티티 해소 접근을 설계할 수 있다.
- 다양한 그래프 데이터 과학·그래프 ML 기법으로 LLM 같은 블랙박스에 과도히 의존하지 않고 KG에서 패턴·통찰을 도출할 수 있다.
