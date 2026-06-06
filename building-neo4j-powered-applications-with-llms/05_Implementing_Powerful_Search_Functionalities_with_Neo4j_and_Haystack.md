# 05. Implementing Powerful Search Functionalities with Neo4j and Haystack

## 챕터 개요 (3줄 요약)

- Haystack(오픈소스 NLP 프레임워크)을 Neo4j와 통합하여 LLM·graph database 기반 AI 검색 시스템을 구축한다.
- OpenAI 임베딩으로 영화 plot embedding을 생성·저장하고 Neo4j vector index로 vector similarity search를 수행한다.
- Gradio로 검색 기반 챗봇 인터페이스를 만들고, 임베딩 모델·Neo4j query·로깅으로 통합을 fine-tuning한다.

---

## 1. Generating embeddings with Haystack to enhance your Neo4j graph

> embedding은 텍스트를 고차원 vector로 변환해 similarity search를 가능케 하며, 단순 키워드 매칭을 넘어 문맥 기반 검색의 정확성을 높인다.

- Haystack v2.5.0과 OpenAI text-embedding-ada-002 모델을 사용해 임베딩을 생성한다.
- initialize_haystack(): InMemoryDocumentStore와 OpenAITextEmbedder를 초기화(.env에 OPENAI_API_KEY 필요, 유료 구독 필수).
- retrieve_movie_plots(): `MATCH (m:Movie) WHERE m.embedding IS NULL`로 embedding 없는 영화의 tmdbId·title·overview를 조회.
- generate_and_store_embeddings(): ThreadPoolExecutor로 병렬 임베딩 생성(max_workers=10).
- store_embedding_in_neo4j(): `MATCH (m:Movie {tmdbId}) SET m.embedding = $embedding`으로 각 Movie node에 embedding property 저장.
- verify_embeddings(): embedding이 저장된 node를 조회해 정상 저장을 검증.

---

## 2. Connecting Haystack to Neo4j for advanced vector search

> embedding 저장 후 embedding property에 vector index를 구성하여 high-dimensional 공간에서 유사 node를 빠르게 검색한다.

### Vector search index 생성 (Neo4j)

- 기존 index를 drop 후 새 vector index 생성, dimensions 1536, similarity_function 'cosine'.

```
CREATE VECTOR INDEX overview_embeddings IF NOT EXISTS
FOR (m:Movie) ON (m.embedding)
OPTIONS {indexConfig: {
  `vector.dimensions`: 1536,
  `vector.similarity_function`: 'cosine'}}
```

### Cypher + Haystack vector search

- OpenAITextEmbedder로 query를 embedding으로 변환 후 `db.index.vector.queryNodes("overview_embeddings", $top_k, $query_embedding)`로 top_k 유사 영화 검색.
- Neo4jDynamicDocumentRetriever를 Haystack Pipeline에 추가하고 query_embedder.embedding을 retriever.query_embedding에 connect.
- 장점: Cypher 유연성(cast·genre·관계 메타데이터까지 조회), enriched 결과, 대규모 graph 최적화.
- 예시: "A hero must save the world from destruction" 질의 → The Matrix(0.98), Inception(0.96), The Dark Knight(0.94) 반환.

```
[User Query]
   -> OpenAITextEmbedder -> [query embedding]
   -> Neo4j vector index (cosine) -> top_k Movie nodes
   -> Cypher enrich (title, overview, cast, genres, score)
```

---

## 3. Building a search-driven chatbot with Gradio and Haystack

> Gradio로 웹 기반 챗봇 인터페이스를 만들어 자연어 query로 Neo4j 영화 embedding의 vector search를 트리거한다.

- gr.Interface로 입력 Textbox(영화 취향)와 출력 Textbox(추천)를 구성하고 예시 query 제공.
- perform_vector_search(): query embedding 생성 후 Neo4jDynamicDocumentRetriever로 유사 영화 검색.
- main()에서 create_or_reset_vector_index() 호출 후 chat_interface.launch()로 실행.
- `python search_chatbot.py` 실행 시 브라우저에 Gradio 인터페이스가 뜨고 "Tell me about a hero who saves the world." 같은 query에 응답.
- 챗봇은 Neo4j에 저장된 embedding으로 vector 기반 검색하여 영화 제목·배우 등 맥락 관련 결과를 반환한다.

---

## 4. Fine-tuning your Haystack integration

> 검색 성능·정확도·사용자 경험을 개선하기 위한 고급 기법을 적용한다.

- 임베딩 모델 실험: 현재 text-embedding-ada-002 사용. text-embedding-3-small(다국어·영어 성능↑, 비용 최대 5배 효율적)이나 text-embedding-3-large 고려 가능.
- Neo4j query 최적화: vector index 외에 title, tmdbId 등 자주 조회하는 property에 추가 index 생성으로 비-embedding 조회 속도 향상.
- Logging·분석: chatbot_queries.log에 user query를 기록·분석하여 indexing 전략·retriever·임베딩 모델을 조정.
- 이러한 조정으로 챗봇이 복잡한 query를 확장적으로 처리하고 더 관련성 높은 결과를 반환한다.

### 배포 환경

- Python 3.11, Neo4j AuraDB/로컬, Haystack(haystack-ai), OpenAI(유료 키), Gradio v5.23.1, Hugging Face 계정, python-dotenv 필요.
- Gradio를 Hugging Face Spaces에 호스팅하여 검색 시스템을 배포한다.

---

## Summary (핵심 정리)

- OpenAI 임베딩으로 Neo4j graph를 강화하고 vector index로 similarity search를 가능케 했다.
- Haystack을 Neo4j에 연결(Neo4jDynamicDocumentRetriever)하고 Gradio 챗봇으로 자연어 영화 검색 경험을 완성했다.
- 다음 Chapter 6에서는 Haystack의 advanced search·검색 최적화와 대규모 graph의 query 최적화를 다룬다.
