# 07. Creating and Connecting a Knowledge Graph to an AI Agent

## 챕터 개요 (3줄 요약)

- 지식 그래프(KG)는 엔티티와 관계를 트리플로 표현해 노이즈를 줄이고 추론을 가능하게 하는 구조다.
- LLM으로 KG를 구축(NER, RE)·평가·정제·보강·배포하고, GraphRAG로 정보를 검색하는 법을 다룬다.
- KG 임베딩, GNN, LLM 그래프 추론과 GraphRAG의 미해결 과제까지 살펴본다.

---

## 1. Introduction to knowledge graphs

> 지식 그래프는 엔티티(노드)를 의미 관계(엣지)로 연결한 그래프로, 지식을 압축적으로 표현하고 추론을 지원한다.

- KG(Knowledge Graph)는 사실 트리플 (head, relation, tail)로 지식 베이스를 표현한 방향 그래프다.
- 그래프 유형: 무방향, 방향, 가중치, 라벨, 멀티그래프.
- KG는 노드(실세계 엔티티), 관계(의미 연결), 속성(properties)을 가진다.
- 그래프는 표·JSON·XML 등 어떤 데이터도 매핑 가능한 보편적 표현이며 병합이 쉽다.
- KGR(Knowledge Graph Reasoning)에서 새 관계 예측과 KG 갱신이 중요한 작업이다.
- 계층적·멀티모달·시간적(temporal) KG 등 다양한 확장이 있다.
- 분류 체계(taxonomy)는 계층 구조, 온톨로지(ontology)는 규칙·속성으로 추론을 지원한다.
- 온톨로지는 도메인 독립형과 도메인형으로 나뉜다.

### KG 트리플 표현

```
fact f = (head, relation, tail)
example: (Napoleon, BornIn, Ajaccio)
G = (E entities, R relations, F facts)
```

---

## 2. Creating a knowledge graph with your LLM

> KG 구축은 생성·평가·정제·보강·배포의 다단계 과정이며 LLM으로 엔티티·관계를 추출한다.

- 지식 생성(knowledge creation)은 온톨로지 정의(top-down/bottom-up)부터 시작한다.
- NER(Named Entity Recognition)은 텍스트에서 엔티티를 추출·분류하는 작업이다.
- RE(Relation Extraction)는 엔티티 간 연결을 식별·분류하는 작업이다.
- 과거 규칙·통계 기반 방식은 확장성이 낮고 라벨이 필요하나, LLM은 ICL로 라벨 없이 추출한다.
- LLM은 한 단계로 NER과 RE를 함께 수행하며, LLM 자체에서 트리플을 증류(distill)할 수도 있다.
- 실습: Neo4j와 LangChain으로 커스텀 프롬프트(Cypher 생성) 또는 LLMGraphTransformer를 사용한다.
- 지식 평가는 정확성·완전성·간결성·시의성·접근성·보안 등 차원으로 측정한다.
- 지식 정제(cleaning)는 구문·온톨로지·의미·사실 오류를 탐지·수정한다.
- 지식 보강(enrichment/completion)은 joint/MLM/separated 인코딩이나 프롬프트로 누락 트리플을 예측한다.
- 호스팅: 관계형 DB, 문서 모델, 그래프 DB(Neo4j/Cypher), 트리플 스토어 중 선택한다.

---

## 3. Retrieving information with a knowledge graph and an LLM

> GraphRAG는 벡터 RAG의 관계 무시·중복·전역 정보 부족 한계를 KG 검색으로 해결한다.

- 벡터 RAG의 한계: 관계 무시, 중복 정보, 전역 정보 부족.
- GraphRAG는 G-indexing, G-retrieval, G-generation 세 단계로 구성된다.
- 그래프 인덱싱은 오픈 KG(Wikidata)나 자체 구축 KG를 사용하며 텍스트·임베딩·하이브리드 인덱싱이 있다.
- 검색 과제: 후보 서브그래프 폭증, 텍스트-그래프 간 유사도 측정.
- 검색기 유형: 비모수(heuristic/k-hop), GNN 기반, LLM 기반 검색기.
- 검색 granularity: 노드, 트리플, 경로(path), 서브그래프, 하이브리드.
- 그래프 변환기로 그래프 정보를 자연어·구문 트리·GraphML 등 LLM이 이해하기 쉬운 형태로 변환한다.
- 응용: 질의응답, 과학 발견, MedGraphRAG(의료), 추천 시스템, 법률·금융.
- HybridRAG는 GraphRAG와 벡터 RAG를 결합한다.

### GraphRAG 3단계

```
G-indexing  : build & index knowledge graph
G-retrieval : query -> extract relevant subgraph
G-generation: subgraph context -> LLM -> answer
```

---

## 4. Understanding graph reasoning

> 그래프 작업은 KG 임베딩, GNN, LLM으로 풀며 이들은 LLM·에이전트와 시너지를 낸다.

- 그래프 구조 이해 작업: 차수 계산, 경로 탐색, 해밀턴 경로, 위상 정렬 등.
- 그래프 학습 작업: 노드 분류, 그래프 분류, KGQA(Knowledge Graph Question Answering).
- KGE(Knowledge Graph Embedding)는 그래프를 저차원 벡터 공간에 투영한다.
- TransE(h+r≈t), RotatE(대칭·구성 패턴), ATTH(쌍곡 공간 계층 구조) 등 방법이 있다.
- GNN(Graph Neural Network)은 메시지 패싱(message passing)으로 이웃 정보를 집계한다.
- GAT(Graph Attention Network)는 이웃별 중요도를 다르게 학습한다.
- LLM 그래프 추론: 수동 프롬프트, 자기 프롬프트(self-prompting), API 호출 프롬프트.
- LLM은 GNN의 노드 특징 임베딩·라벨 생성에 활용되어 시너지를 낸다.
- 그래프 형태 추론: GoT(Graph of Thoughts, think on graph), verify on graph.

---

## 5. Ongoing challenges in knowledge graphs and GraphRAG

> 대규모 KG의 확장성, GraphRAG 최적화, LLM의 구조적 데이터 이해 부족이 주요 과제다.

- 대규모 KG는 표현력과 연산 효율의 균형, 최적화된 쿼리 알고리즘이 필요하다.
- KG는 본질적으로 불완전해 동적 갱신·완성 파이프라인이 필요하다.
- 멀티모달 KG 통합은 추론을 개선하나 관리 복잡도를 크게 높인다.
- GraphRAG는 벤치마크 표준 부재, 무손실 압축 미해결, 추상형 QA 약점이 있다.
- LLM은 다음 단어 예측으로 학습되어 구조적·수치적 데이터(표·그래프) 이해가 약하다.
- 그래프 데이터 SFT(Supervised Fine-Tuning)는 작은 LLM도 큰 LLM보다 나은 성능을 낸다.
- 인간은 경험으로 공간 관계를 학습하나 LLM은 멘탈 맵이 없어 추상적 공간 개념에 불리하다.

---

## Summary (핵심 정리)

- 텍스트의 핵심인 엔티티와 관계를 지식 그래프로 표현하는 환원주의적 접근을 배웠다.
- LLM과 KG의 상호작용(추출, 추론, 컨텍스트 보강)과 GraphRAG를 익혔다.
- 벡터 RAG와 GraphRAG의 통합(HybridRAG)이 미래 방향이며, 다음 장에서 강화학습과 에이전트를 다룬다.
