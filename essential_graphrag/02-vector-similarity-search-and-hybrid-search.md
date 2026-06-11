# 02. Vector Similarity Search and Hybrid Search

## 챕터 개요 (3줄 요약)
- RAG는 retriever(관련 정보 검색)와 generator(응답 생성) 두 컴포넌트로 구성된다.
- embedding model + vector index + cosine similarity로 unstructured text에 대한 vector similarity search를 구현한다.
- full-text search를 더해 hybrid search를 구성하면 keyword 매칭까지 보완해 retriever 품질이 향상된다.

---

## 1. Components of a RAG architecture
> retriever는 vector similarity search로 관련 chunk를 찾고, generator는 그 정보로 응답을 생성한다.

- **Retriever 구성요소**:
  - **vector index**: 유사 벡터 검색을 빠르게 하는 자료구조. approximate nearest neighbor 검색 → 속도 vs 정확도 tradeoff.
  - **similarity 함수**: cosine similarity(의미 유사도, 0~1) 권장, Euclidean distance도 존재.
  - **embedding model**: 텍스트를 vector(embedding)로 변환. 전체 파이프라인에서 동일 모델 유지 필수(바꾸면 index 재생성). embedding dimension이 클수록 정보량↑·비용↑.
  - **text chunking**: 텍스트를 작은 조각으로 분할 → 더 좁고 specific한 embedding → retrieval 정확도↑. 분할 단위·크기·sliding window는 use case별 실험 필요.
- **Generator**: 보통 LLM. retriever가 지식을 제공하므로 작고 빠른 모델 사용 가능. "지식"이 아니라 "텍스트 생성"용으로 쓰여 hallucination↓.

---

## 2. RAG using vector similarity search
> 데이터 셋업(chunk→embed→vector index 저장)과 query 시점(질문 embed→검색→LLM 생성) 두 단계로 동작한다.

- 데이터 셋업 필요 요소: text corpus, chunking 함수, embedding model, vector search 가능한 DB.
- 예제: "Einstein's Patents and Inventions" 논문을 corpus로 사용.
- **chunking**: sliding window size 500자 + overlap 40자, 단어 깨짐 방지 위해 공백에서만 분할.
- **embedding**: OpenAI `text-embedding-3-small` (1536 dim). 대안: Hugging Face `all-MiniLM-L12-v2`(로컬 CPU 가능).
- **DB**: Neo4j 사용. 데이터 모델 = `:Chunk` 노드(text, embedding 속성).
- vector index 생성 시 dimension(1536) 명시 → embedding 모델 변경 시 index 재생성.
- **검색**: 질문을 동일 모델로 embed → `db.index.vector.queryNodes`로 top-k 유사 chunk 반환(score 포함).
- **생성**: system message(역할·제약) + user message(검색 chunk + 질문)를 LLM에 전달해 답변 생성.

```cypher
CALL db.index.vector.queryNodes('pdf', 2, $question_embedding)
YIELD node AS hits, score
RETURN hits.text AS text, score, hits.index AS index
```

---

## 3. Adding full-text search to enable hybrid search
> vector search와 full-text(keyword) search를 결합·정규화해 합치면 retriever 결과가 더 좋아진다.

- **full-text search**: vector space 유사도가 아닌 keyword 정확 매칭. exact match 필요.
- Neo4j full-text index 생성 후 hybrid 수행.
- **hybrid search 원리**: vector search + full-text search 각각 수행 → 각 검색의 max score로 나눠 score 정규화 → UNION 후 중복 제거 → top-k 반환.
- 효과: vector가 놓친 매칭을 keyword가 보완(예제에서 2위 결과가 달라짐).

```cypher
CALL { /* vector */ ... RETURN node, (score/max) AS score
  UNION /* fulltext */ ... RETURN node, (score/max) AS score }
WITH node, max(score) AS score ORDER BY score DESC LIMIT $k
RETURN node, score
```

---

## 4. Concluding thoughts
> hybrid search는 unstructured 데이터만으로는 여전히 품질·정확도 한계가 있어 structure 도입이 필요하다.

- vector + full-text 결합이 단일 방식보다 낫다.
- 그러나 unstructured 데이터 기반이라 reference 누락, 맥락 부족 한계 존재.
- 다음 장: retriever 개선 전략.

---

## Summary (핵심 정리)
- RAG는 retriever와 generator로 구성되며, retriever가 관련 정보를 찾고 generator가 응답을 만든다.
- text embedding은 의미를 vector space로 표현해 vector similarity search를 가능케 한다.
- full-text search를 더하면 hybrid search로 retriever 성능을 보완할 수 있다.
- vector·hybrid search는 유효하나, 데이터 복잡도가 커질수록 품질·정확도 한계가 드러난다.
