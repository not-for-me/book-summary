# 02. Demystifying RAG

## 챕터 개요 (3줄 요약)

- LLM의 환각(hallucination) 원인과 완화 기법을 살펴보고, 그 근본 해법으로 RAG(Retrieval-Augmented Generation)를 제시한다.
- RAG 아키텍처(retriever + encoder-decoder)를 코드 예제로 해체하여 일반 LLM 흐름과의 차이를 설명한다.
- vector similarity search, keyword matching(BM25), passage retrieval 등 검색 기법과 end-to-end RAG 파이프라인 구현을 다룬다.

---

## 1. Understanding the power of RAG

> RAG는 2020년 Meta(FAIR) 연구진이 제안한 프레임워크로, 모델 학습에 포함되지 않은 외부 데이터를 활용해 출력을 향상시킨다.

- LLM 환각의 대표 사례: ChatGPT가 만든 가짜 판례를 제출해 벌금을 문 뉴욕 로펌(Avianca 소송) 사건.
- 환각 원인: overfitting(학습 데이터 통계 패턴 과적합), causal reasoning 부재, temperature 설정(0~1, 높을수록 창의성↑·환각↑), 정보 누락, 편향된 학습 데이터.
- 완화 기법: prompt engineering(구체적 지시), in-context learning(few-shot prompting, 예시 포함), fine-tuning(특정 데이터셋 추가 학습).
- RLHF(Reinforcement Learning with Human Feedback): 인간 평가자가 출력을 점수화하여 모델 행동을 조정, 환각 감소에 특히 유효.
- 위 기법들도 도메인 지식 기반의 정확·설명가능한 GenAI 앱을 빠르게 만들기엔 부족하며, 그 해법이 grounding 기반의 RAG다.
- RAG는 neural retriever와 sequence-to-sequence generator를 결합한 hybrid 아키텍처로, 외부 knowledge base에서 문서를 검색해 context로 제공한다.
- RAG는 모든 지식을 파라미터에 담지 않고 동적 검색하므로 모델 크기를 줄이면서 정확도·확장성을 유지한다.

---

## 2. Deconstructing the RAG flow

> 일반 LLM 흐름과 달리 RAG는 LLM 호출 전에 context를 제공하는 중간 데이터 소스를 둔다.

- 일반 LLM 흐름(Figure 2.2): 사용자 prompt → LLM API → LLM 응답 생성 → 사용자. 중간 과정이 없다.
- RAG 흐름(Figure 2.3): prompt → RAG 모델(retriever + encoder-decoder) → knowledge repository 검색 → context 결합 → 응답.
- Retriever: 비구조화 문서·passage 또는 표·knowledge graph 같은 구조화 데이터를 포함한 저장소에서 가장 관련 있는 정보를 찾는다.
- DPR(Dense Passage Retrieval) 예시: 문서를 embedding으로 인코딩 후 cosine similarity로 top 결과를 검색한다.
- Encoder-decoder/augmented generation: encoder가 prompt와 검색 정보를 종합 표현으로 만들고, decoder가 응답을 생성(예: T5 모델 + beam search decoding).
- Beam search decoding: greedy와 달리 여러 후보 sequence(beam)를 동시에 탐색해 고품질 결과 확률을 높인다.

### RAG 파이프라인 구조

```
[User Query]
     |
     v
[Retriever] --search--> [Knowledge Repository: docs / KG / tables]
     |  (top relevant docs)
     v
[Encoder-Decoder (e.g. T5)] --generate--> [Grounded Response]
```

---

## 3. Retrieving external information for your RAG

> RAG 성공은 방대한 외부 knowledge base에서 관련 정보를 검색하는 능력에 달려 있다. 기법은 크게 세 범주로 나뉜다.

### Vector similarity search

- 입력 query를 embedding으로 변환하고, 문서 embedding과 dot product로 유사도를 계산해 정렬·반환한다.
- 유사한 텍스트는 유사한 embedding을 가진다는 원리로, 정확 키워드가 달라도 개념적 유사성을 포착한다.
- 예: "solar energy" 질의 시 관련 문서 상위 랭크(예: "Graph databases like Neo4j are used to model complex relationships." 같은 문서도 점수화됨).

### Keyword matching (BM25)

- BM25는 term frequency와 document length를 고려하는 확률적 keyword 기반 검색 함수(BM25Okapi, k1=1.5, b=0.75).
- 효율적·해석가능하지만 동의어나 문맥 의미를 인식하지 못해 noise에 취약하다.
- vector search(DPR)는 의미적 유사성에, BM25는 정확한 단어 매칭과 explainability가 중요한 작업에 적합하다.

### Passage retrieval

- 문서 전체가 아닌 query에 직접 답하는 특정 passage를 추출하여 더 정밀한 정보 추출을 수행한다.
- vector search로 문서를 랭킹한 뒤 reader 모델이 word·span 수준에서 relevance를 평가(relevance logit/softmax)한다.
- retriever(의미 유사성) + reader(문맥 정렬)의 dual-stage로 매우 타겟화된 응답을 생성한다.

### Integrating the retrieved information

- 여러 검색 passage를 query와 결합해 단일 입력으로 만들고 T5 같은 생성 모델에 전달하여 통합·풍부한 응답을 합성한다.
- 단순 선택/랭킹을 넘어 여러 소스 정보를 종합(synthesis/summarization)하는 질의에 효과적이다.

---

## 4. Building an end-to-end RAG flow

> 하드코딩 문장 대신 실제 데이터셋(GitHub issues dataset)으로 전체 RAG 흐름을 완성한다.

- Hugging Face datasets로 GitHub issues를 로드하고 pull request 제외·댓글 있는 issue만 필터링한다.
- 필요한 컬럼(title, body, html_url, comments)만 유지하고 pandas DataFrame으로 변환 후 댓글을 행 단위로 explode한다.
- title·body·comments를 concatenate하여 document text를 만들고, sentence-transformers/all-MiniLM-L6-v2 모델로 embedding을 생성·저장한다.
- 질의("How can I load a dataset offline?")의 embedding과 cosine similarity로 semantic search하여 top 5 관련 댓글을 반환한다.
- 이 흐름이 후속 장의 full end-to-end RAG 구현(Neo4j 연동 포함)의 기초가 된다.

### 사용 라이브러리 (기술 요구사항)

- Transformers, PyTorch, scikit-learn, NumPy, SentencePiece, rank_bm25, datasets, pandas, faiss-cpu, Accelerate.
- Python 3.6 이상과 deep learning 기본 개념이 필요하며, GitHub 저장소(ch2)에서 전체 코드를 clone할 수 있다.

---

## Summary (핵심 정리)

- RAG는 LLM 환각을 외부 검증 지식으로 grounding하여 해결하며, retriever·encoder·decoder 구성요소로 동작한다.
- vector similarity search, BM25 keyword matching, passage retrieval 세 검색 기법이 외부 지식을 효과적으로 활용한다.
- 다음 Chapter 3에서는 graph data modeling과 Neo4j로 knowledge graph를 만드는 방법으로 이어진다.
