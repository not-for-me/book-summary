# 05. Extending Your Agent with RAG to Prevent Hallucinations

## 챕터 개요 (3줄 요약)

- RAG(Retrieval-Augmented Generation)는 외부 메모리에서 컨텍스트를 찾아 LLM의 환각과 지식 노후화를 줄인다.
- 청킹, 임베딩, 벡터 데이터베이스, 출력 평가, RAG와 파인튜닝 비교 등 구성 요소를 상세히 다룬다.
- 영화 추천 에이전트 실습으로 인코더, Chroma 벡터 DB, Mistral LLM을 결합한 전체 파이프라인을 구축한다.

---

## 1. Exploring naïve RAG

> 정보 검색은 쿼리와 관련 문서를 매칭해 관련도 순으로 반환하며, RAG에서는 트랜스포머가 검색 엔진 역할을 한다.

- TF-IDF나 BM25 같은 희소 벡터로 코사인 유사도를 계산해 문서를 랭킹할 수 있다.
- BM25는 문서 길이 정규화(b)와 TF 포화(k) 두 파라미터를 더한 TF-IDF 변형이다(권장 b=0.75, k=1.2~2).
- BM25는 의미를 못 잡고 다의어·어휘 불일치(vocabulary mismatch) 문제가 있다.
- 밀집 벡터(dense vector)는 트랜스포머로 문맥 정보를 담아 단어 의미를 명확히 한다("bank" 사례).
- 이상치 차원(rogue dimensions) 때문에 비등방성(anisotropy)이 생겨 z-score 정규화로 완화한다.
- 단일 인코더(single encoder)와 바이 인코더(bi-encoder) 두 전략이 있으며, 바이 인코더는 빠르나 덜 정확하다.
- 환각(hallucination)은 LLM이 특정 정보(날짜·수치·희귀 정보) 보존에 약하기 때문에 발생한다.
- RAG(2020, Meta)는 파라미터 메모리 외에 외부(nonparametric) 메모리를 활용하며 인덱싱·검색·생성 3단계로 구성된다.

### RAG 3단계 흐름

```
Indexing: raw data -> text -> chunks -> embed -> vector DB
Retrieval: query -> embed -> similarity -> top-k chunks
Generation: prompt(query + chunks + history) -> LLM -> answer
```

---

## 2. Retrieval, optimization, and augmentation

> 청킹, 임베딩, 벡터 DB 선택은 RAG 성능을 좌우하며 각 단계에서 실무적 선택이 필요하다.

- 청크가 너무 작으면 문맥을 잃고, 너무 크면 비특이적이며 컨텍스트 길이를 초과하면 절단(truncation)된다.
- 고정 길이 청킹(문자/토큰 기반)은 가장 단순하며 슬라이딩 윈도우로 중첩(overlap)을 둘 수 있다.
- 컨텍스트 인식 청킹은 정규식으로, 재귀적(recursive)·계층적(hierarchical) 청킹은 텍스트 구조를 존중한다.
- 의미 청킹(semantic chunking): k-means, 명제(proposition) 기반, 통계적 병합(statistical merging).
- 밀집 인코더는 BERT 기반 바이 인코더가 흔하며 대조 학습(contrastive learning)으로 학습한다.
- MultiNLI(Multi-Genre Natural Language Inference) 코퍼스로 entailment(긍정)·contradiction(부정) 예시를 만든다.
- 손실 함수: 코사인 유사도 손실, 다중 부정 랭킹 손실(InfoNCE).
- 임베딩 최적화: 이진 양자화(binary quantization, 최대 32배 절감), int8 변환, Matryoshka Representation Learning.
- 벡터 DB는 ANN(Approximate Nearest Neighbors), HNSW(Hierarchical Navigable Small World) 알고리즘으로 검색을 가속한다.

---

## 3. Evaluating the output

> 검색 품질은 정밀도·재현율 기반 지표로, 생성 품질은 LLM을 심판으로 쓰는 지표로 평가한다.

- 정밀도(precision)는 검색된 문서 중 관련 비율, 재현율(recall)은 관련 문서 중 검색된 비율이다.
- 랭킹을 고려하려면 정밀도-재현율 곡선(precision-recall curve)을 사용한다.
- MAP(Mean Average Precision)는 관련 항목 위치별 정밀도의 평균을 쿼리 전체로 평균낸다.
- MRR(Mean Reciprocal Rank)은 질의응답에서 첫 관련 항목 순위의 역수 평균이다.
- LLM 심판 지표: 충실성(faithfulness), 컨텍스트 재현율/정밀도/관련성, 엔티티 재현율.
- 답변 정확성(answer correctness), 요약 점수, 답변 관련성, 유창성(fluency), 일관성(coherence)도 평가한다.
- 이들 지표는 통계값이 아니라 인간이나 LLM 평가자의 비판적 평가를 요구한다.

---

## 4. Comparison between RAG and fine-tuning

> RAG와 파인튜닝은 모두 미학습 지식을 제공하지만 갱신성, 커스터마이징, 해석성에서 차이가 있다.

- RAG는 실시간 동적 지식 갱신이 가능하나 파인튜닝은 정적이라 재학습이 필요하다.
- RAG는 데이터 처리가 최소이나 파인튜닝은 양질의 데이터셋이 필요하다.
- RAG는 행동·문체를 바꾸지 않으나 파인튜닝은 행동·문체·새 기술을 바꾼다.
- RAG는 출처 추적으로 해석성이 높고 환각에 덜 취약하다.
- 요약은 파인튜닝, 질의응답은 RAG, 코드 생성은 둘 다 유익하다.
- 둘은 대립이 아니라 RAG + LLM/인코더 파인튜닝으로 함께 쓸 수 있다.
- 인코더 파인튜닝은 처음부터 학습보다 저렴하며 Sentence Transformer로 쉽게 수행한다.

---

## 5. Using RAG to build a movie recommendation agent

> LLM, 인코더/검색기, 벡터 DB를 결합해 자연어 질문으로 영화를 추천하는 RAG 시스템을 만든다.

- 문서를 NLTKTextSplitter(청크 크기 1,500자)로 의미를 보존하며 청킹한다.
- all-MiniLM-L6-v2(2,270만 파라미터)를 임베더로 사용해 빠르게 청크를 벡터화한다.
- Chroma 벡터 DB 클라이언트를 시작하고 컬렉션을 만들어 메타데이터(영화 제목)와 함께 청크를 저장한다.
- 추론 시 쿼리를 같은 임베더로 벡터화하고 top-k 유사 문서를 검색한다.
- Mistral-7B-Instruct LLM에 컨텍스트와 질문을 담은 [INST] 프롬프트를 제공해 답변을 생성한다.
- retrieve_documents로 청크·제목을 얻고 generate_answer로 최종 답을 생성한다.
- 이 원리는 모든 문서 코퍼스에 동일하게 적용된다.

### 영화 추천 RAG 파이프라인

```
query -> embedder -> Chroma DB (top-k search)
  -> retrieved chunks + titles -> prompt -> Mistral-7B -> answer
```

---

## Summary (핵심 정리)

- RAG는 환각을 줄이는 가장 빠르게 성장하는 패러다임이며 파인튜닝 대비 여러 장점이 있음을 배웠다.
- naïve RAG는 LLM, 임베더, 벡터 데이터베이스 세 핵심 요소로 구성됨을 익혔다.
- 다음 장에서는 RAG의 진화된 고급 구성 요소와 파라미터 메모리·컨텍스트의 상호작용을 다룬다.
