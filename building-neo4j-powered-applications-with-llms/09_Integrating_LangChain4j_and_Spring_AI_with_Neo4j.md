# 09. Integrating LangChain4j and Spring AI with Neo4j

## 챕터 개요 (3줄 요약)

- LangChain4j 또는 Spring AI로 H&M graph를 augment하여 고객 구매 요약·embedding을 생성하는 knowledge graph를 구축한다.
- 두 Java 프레임워크로 동일한 graph augmentation 앱(GraphRAG 방식)을 구현하고 API·통합 방식의 차이를 비교한다.
- 생성된 embedding에 vector index를 만들어 유사 article·유사 customer를 찾아 개인화 추천을 제공한다.

---

## 1. Setting up LangChain4j and Spring AI

> LangChain4j와 Spring AI는 같은 작업을 수행하는 대안이며, 둘 중 하나만 있으면 GenAI 프로젝트 구축이 가능하다.

- spring initializr(start.spring.io)로 Maven·Java 17 스타터 프로젝트 생성.
- LangChain4j 프로젝트: Spring Web만 추가, LangChain4j 의존성은 수동 추가.
- Spring AI 프로젝트: Spring Web, OpenAI, Neo4j Vector Database 의존성 추가.
- 기술 요구: Maven, Java 17, IntelliJ, Spring Boot, Neo4j Desktop(APOC 5.21.2, GDS 2.9.0 플러그인).

### 앱이 하는 일 (augmentation 접근)

- 분석할 season 선택(예: SUMMER_2019~FALL_2019 사이 거래로 구매 행동 이해).
- 해당 거래의 article을 구매 순서대로 retrieve.
- LLM으로 구매 내역 요약(구매 순서 보존) 생성.
- OpenAI LLM으로 요약 텍스트의 embedding 생성 후, 해당 season relationship(예: SUMMER_2019)에 저장.

---

## 2. Building your recommendation engine with LangChain4j

> Neo4j graph retriever로 조건에 맞는 transaction chain을 가져오고, LLM으로 요약·embedding을 생성하는 GraphRAG 앱을 구현한다.

- pom.xml에 의존성 추가: langchain4j-spring-boot-starter, langchain4j-open-ai-spring-boot-starter, langchain4j-neo4j, langchain4j-embeddings-all-minilm-l6-v2.
- application.properties 설정: OpenAI chat-model(gpt-4o-mini), embedding-model(text-embedding-3-large), Neo4j(bolt://localhost:7687, database=hmreco, batchSize=5).

### Neo4j integration

- @ConfigurationProperties(prefix="neo4j")로 Neo4jConfiguration bean이 설정값을 읽음.
- Neo4jService: setup()으로 Driver 연결, getDataFromDB(start, end)로 season 거래의 elementId·article 설명을 순서대로 조회(color 등 속성 포함).
- saveEmbeddings(): `db.create.setRelationshipVectorProperty(r, 'embedding', ...)`로 relationship에 embedding·summary 저장.
- saveArticleEmbeddings(): `db.create.setNodeVectorProperty(a, 'embedding', ...)`로 Article node에 embedding 저장.

### OpenAI chat integration (@AiService)

- @AiService 인터페이스 ChatAssistant에 @SystemMessage로 Role(패션 전문가)·Goal(2섹션 요약: 전체 선호 3문장 + 개별 구매 3~5개)·{text} 데이터 변수 정의.
- OpenAIChatService.getSummaryText()가 assistant.chat() 호출 — LangChain4j Spring이 구현을 자동 제공.

### OpenAI embedding integration

- OpenAIEmbeddingModelService: EmbeddingModel을 생성자 주입, generateEmbedding(text)으로 Response<Embedding> 반환.

### Final application (REST endpoint)

- LangchainGraphAugmentController: `/augment/{startSeason}/{endSeason}`로 augmentation 시작(UUID 반환), `/augmentArticles`로 article augmentation, `/augment/status/{requestId}`로 진행률 조회.
- ProcessRequest(Runnable): graph에서 데이터 조회 → 각 record마다 LLM 요약 → embedding 생성 → batch(설정 크기)로 Neo4j 저장. 장시간 소요되어 dump는 약 10,000 고객만 커버.
- ProcessArticles: article 텍스트를 batch 모드로 embedding 생성(단건보다 빠름).

---

## 3. Building your recommendation engine with Spring AI

> LangChain4j와 유사하게 GraphRAG로 graph를 augment하되, Spring AI 방식의 API 차이를 보인다.

- 의존성: Spring starter에서 모두 추가되어 별도 수정 불필요.
- application.properties: spring.ai.openai.api-key, spring.ai.openai.embedding.options.model=text-embedding-3-large, Neo4j 설정.
- Neo4j integration: LangChain4j와 동일.
- OpenAI chat integration: ChatClient를 직접 초기화(추상화되지 않음). Role·Goal을 system prompt template으로, data를 user message로 분리하여 `.prompt().system(...).user(...).call().chatResponse()` 호출.
- Embedding integration: EmbeddingModel 주입, embed(text)/embed(textList)로 단건·batch 임베딩.
- Final application: LangChain4j와 동일한 흐름·코드(패키지명만 다름).

### 두 프레임워크 차이

- LangChain4j: @AiService로 chat을 고수준 추상화, single system message에 role·goal·data 통합.
- Spring AI: ChatClient를 직접 사용, role/goal은 system prompt, data는 user message로 분리. 결과는 동일.

---

## 4. Fine-tuning your recommendation system

> augmentation 실행 후 vector index를 만들어 유사 article·customer 기반 추천을 제공한다.

- 실행: IDE에서 앱 실행 후 `http://localhost:8080/augment/SUMMER_2019/FALL_2019` 호출(UUID 반환), `/augment/status/{uuid}`로 진행률 확인, `/augmentArticles`로 article embedding 생성.

### Vector index 생성

```
CREATE VECTOR INDEX `article-embeddings` IF NOT EXISTS
FOR (a:Article) ON a.embedding
OPTIONS { indexConfig: { `vector.dimensions`: 3072, `vector.similarity_function`: 'cosine' }}

CREATE VECTOR INDEX `summer-2019-embeddings` IF NOT EXISTS
FOR ()-[r:SUMMER_2019]->() ON (r.embedding)
OPTIONS { indexConfig: { `vector.dimensions`: 3072, `vector.similarity_function`: 'cosine' }}
```

- text-embedding-3-large는 3072 차원, cosine similarity 사용.

### 유사 검색 활용

- 유사 article: `db.index.vector.queryNodes('article-embeddings', 5, a.embedding)` — 자기 자신 score 1.0, 유사 beach dress들이 0.87~0.88. score가 0.9에 가까우면 매우 유사.
- 유사 customer: `db.index.vector.queryRelationships('summer-2019-embeddings', 5, r.embedding)`로 구매 요약 embedding이 유사한 고객 조회(예: 선명한 색·swimwear 선호 고객끼리 매칭).
- 추천 생성: 유사 고객의 SUMMER_2019~FALL_2019 구매 article을 모으고, `apoc.coll.subtract`로 원 고객이 안 산 article을 빼서 10개 추천.

```
[Customer] -[:SUMMER_2019 {embedding}]-> ...
   queryRelationships -> top-K 유사 고객
   유사 고객 구매 article 수집 - 원 고객 구매 = 추천 article
```

- 구매 순서를 포착하지만, 요약을 embedding이 어떻게 담느냐가 유사 고객 판정을 좌우한다.
- "누가 유사 고객인가"를 직접 정하는 방식의 한계는 다음 장의 GDS로 개선한다.

---

## Summary (핵심 정리)

- LangChain4j와 Spring AI로 H&M transaction graph를 LLM chat·embedding으로 augment했다.
- vector index로 유사 article·유사 customer를 찾아 구매 행동 기반 개인화 추천을 구현했다.
- 다음 Chapter 10에서는 Graph Data Science 알고리즘으로 추천을 더 강화하는 방법을 다룬다.
