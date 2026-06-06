# 07. Introducing the Neo4j Spring AI and LangChain4j Frameworks for Building Recommendation Systems

## 챕터 개요 (3줄 요약)

- 추천 시스템과 personalization의 중요성, 전통적 rule-based 접근의 한계를 설명한다.
- 지능형 애플리케이션 구축을 위한 Neo4j의 확장 기능(scalability, security, GDS, vector index 등)을 소개한다.
- Java 기반 프레임워크인 LangChain4j와 Spring AI를 소개하고, Neo4j GenAI 생태계의 GraphRAG 추천 아키텍처를 개관한다.

---

## 1. Understanding extended Neo4j capabilities to build intelligent applications

> knowledge graph는 검색뿐 아니라 personalized recommendation의 견고한 기반이 되며, Neo4j의 DB 기능이 더 나은 앱 구축을 돕는다.

- Scalability: sharding으로 federated graph를 만들어 대규모 데이터셋을 처리하고 비용을 최소화하며 확장.
- Security: role 기반으로 read/write 권한 등 고수준·세분화된 데이터 보안 제어(사용자별로 graph의 다른 부분 조회 가능).
- Flexible deployment: clustering 아키텍처로 수평 확장, read 지역화, 데이터 증가에도 소유 비용 최소화.
- Graph Data Science(GDS) 알고리즘: pathfinding, node similarity, centrality, community detection, link prediction, node classification 등으로 숨은 insight 발굴.
- Vector indexes: embedding을 인덱싱해 유사 node 조회 후 graph traversal로 더 정확한 결과 제공.

---

## 2. Personalizing recommendations

> 추천 시스템은 사용자의 구매·검색 선호에 기반해 제품을 추천하며, 제품 배치뿐 아니라 의료 진단·치료에도 활용된다.

- 개인화 전략:
  - 사용자 프로필 구축: 거래 순서·이벤트 결과·나이·인종·성별 등 행동 패턴으로 사용자를 세그먼트화.
  - 맥락 지원 제공: 마지막 구매 제품이나 현재 치료 수준·증상에 기반한 다음 제품/약물 추천.
  - self-service 경험: 사용자가 추천 고려 특성을 변경해 시스템 반응을 조정.
  - 피드백 반영: 긍정·부정 피드백을 반영해 개별 사용자 요구에 적응.
- 장점: 현재 조회 기반 다음 제품 제안, 행동 기반 인센티브, 브랜드 평판 향상, 환자 치료 regimen 최적화, 공급망 개선, 최적 배송 경로 결정 등.

### Limitations of traditional approaches (rule-based)

- Rule-based 시스템은 입력 데이터에 규칙 집합을 실행해 결정한다(예: 특정 지역 $1,000 초과 카드 거래 자동 거부).
- Static rules(수동 구성, 빠른 응답·저자원)와 Dynamic rules(정교한 rule engine, 결정 트리 상태 의존)로 나뉜다.
- 장점: 일관성(consistency), 확장성(scaling), 효율성(efficient), 유지보수·관리 용이.
- 한계: 복잡성 증가, 경직성(rigidness, 새 데이터·시나리오 적응 어려움), 비즈니스 요구 적응성 부족.
- 진화하는 비즈니스 요구에 rule-based는 한계가 있어, 새 데이터·복잡성에 적응하는 지능형 앱이 필요하다.

---

## 3. Introducing Neo4j's LangChain4j and Spring AI frameworks

> 지능형 추천 시스템을 위해 Neo4j 주변의 Java 프레임워크인 Spring AI와 LangChain4j를 활용한다(Java는 Python보다 빠르고 web 앱·Spring에 강점).

### LangChain4j

- Python LangChain에서 영감받아 Java로 LLM 앱을 구축하는 프레임워크로, LangChain·Haystack·LlamaIndex 개념을 혼합.
- Unified APIs: OpenAI, Google Gemini 등 LLM provider와 Neo4j, Pinecone, Milvus 등 vector store의 복잡성을 통합 API로 은닉.
- Comprehensive toolbox: prompt template, chat memory, AI service, RAG 등 ready-to-use 패키지 제공.
- 주요 기능: 15+ LLM provider, 20+ vector store, AI services(LLM·vector store·embedding·RAG 파이프라인 고수준 API), RAG(Easy RAG 포함).

### Spring AI

- LangChain4j·LlamaIndex에서 영감받아 Spring Framework에 최적화 — Spring 숙련자가 LLM 앱을 빠르게 개발.
- 기능: LLM prompt templates, embedding models(설정 기반 통합), vector stores(Neo4j·Pinecone·Milvus 등 설정 기반 연결), RAG(prompt·embedding·vector store 체이닝).

### Why Java-based frameworks?

- 많은 앱이 Java를 사용하며, 이 프레임워크들은 Amazon Bedrock, Azure OpenAI, Google Gemini, Hugging Face, OpenAI 등 다중 LLM provider와 Neo4j 등 vector store를 지원.
- Neo4j와 결합해 graph feature(path 등)의 embedding을 생성하고, similarity·community detection으로 node를 세그먼트화하여 next-level 추천의 기반을 마련.

---

## 4. Overview of an intelligent recommendation system in Neo4j GenAI ecosystem

> LLM/RAG 원리로 구축된 추천 시스템이 Neo4j GenAI 생태계에서 동작하는 방식(Figure 7.1, Neo4j RAG recommendation architecture).

- Spring AI 앱으로 graph를 augment하여 더 개인화된 추천을 제공.
- RAG에서 vector index와 graph traversal을 함께 활용해 응답을 augment하는 GraphRAG 개념으로 두 세계의 장점을 결합.
- knowledge graph는 더 정확한 응답, 풍부한 context, explainability를 제공하며, Neo4j는 LangChain4j·Spring AI에서 vector store이자 graph database로 LLM 응답을 augment.

```
[User] -> [Spring AI / LangChain4j App]
              |-- vector index (embedding similarity)
              |-- graph traversal (relationships)
              v
        [Neo4j Knowledge Graph] --GraphRAG--> [Grounded, personalized recommendation]
```

---

## Summary (핵심 정리)

- Neo4j의 scalability·security·GDS·vector index 기능이 rule-based를 넘는 적응형 지능형 추천 시스템을 가능케 한다.
- Java 프레임워크 LangChain4j와 Spring AI는 LLM·embedding·vector store·RAG를 통합 API로 제공하여 Neo4j 기반 앱 개발을 단순화한다.
- 다음 Chapter 8에서는 H&M 데이터셋으로 personalized recommendation용 graph data model을 구축하고, Chapter 9에서 Spring AI·LangChain4j와 통합한다.
