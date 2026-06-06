# 08. Constructing a Recommendation Graph with H&M Personalization Dataset

## 챕터 개요 (3줄 요약)

- 실제 대규모 H&M Personalization 데이터셋으로 추천 엔진용 knowledge graph data model을 구축한다.
- customer·article·transaction 데이터를 Neo4j에 로드하되, transaction을 시간 차원의 transaction train(체인)으로 모델링한다.
- consumption 기반 모델링(label·node 정규화)과 seasonal relationship 후처리로 graph에 맥락을 추가해 추천을 최적화한다.

---

## 1. Modeling the recommendation graph with the H&M personalization dataset

> 좋은 graph data model은 RAG의 retrieval을 더 효과적으로 만들며, transaction을 시간 차원의 체인으로 모델링하면 매우 효율적·고성능 검색이 가능하다.

- 2022년 H&M이 Kaggle 대회용으로 공개한 데이터셋: 과거 거래 + customer·product 메타데이터(garment type, 나이, 제품 설명 텍스트, 이미지 등).
- 데이터 구성:
  - images/: article_id별 이미지(graph에 비효율적이라 미사용).
  - articles.csv: product family·color·style·section·department 등 article 메타데이터.
  - customers.csv: customer ID, 나이, fashion news 빈도, active flag, 클럽 멤버 상태, 우편번호.
  - transactions_train.csv: 거래일·article ID·customer ID·가격·sales channel(거래별 unique ID 없음).
- transaction은 이벤트의 sequence이므로 sequence로 모델링하는 것이 합리적이며, Neo4j는 이를 relationship으로 순차 연결된 graph로 저장 — 데이터 지식을 graph에 persist하여 knowledge graph 생성.

### Loading the customer data

> Customer node에 age만 property로 두고 나머지 속성은 label로 매핑하는 consumption 기반 모델링을 적용한다.

- UNIQUE constraint: Customer.id, PostalCode.code 생성.
- LOAD CSV로 IN TRANSACTIONS OF 1000 ROWS 배치 로드. fashion_news_frequency, club_member_status 등을 FN_REGULAR, CLUB_ACTIVE, CLUB_PRE_CREATE, INACTIVE 등 label로 부여.
- PostalCode를 node로 만들어 `(c)-[:LIVES_IN]->(p)` 연결 — label 기반 조회가 property+index보다 저장·lookup 비용 면에서 효율적이고 graph 표시도 직관적.

### Loading the article data

> article의 대부분 속성을 별도 node로 변환하여 graph 내 데이터를 정규화한다.

- UNIQUE constraint 다수 생성: Product, Article, ProductType, ColorGroup, ProductGroup, GraphicalAppearance, PerceivedColor, Department, Section, GarmentGroup, Index, IndexGroup.
- 관계: `(a)-[:OF_PRODUCT]->(p)`, `(p)-[:HAS_TYPE]->(pt)`, `(p)-[:HAS_GROUP]->(pg)`, HAS_GRAPHICAL_APPEARANCE, HAS_COLOR, HAS_PERCEIVED_COLOR, HAS_DEPARTMENT, HAS_INDEX, HAS_SECTION, HAS_GARMENT_GROUP 등.
- 값 중복 없이 정규화된 데이터를 graph에 persist.

### Loading the transaction data (transaction train)

> 거래 순서가 보존된 CSV를 sequence로 로드하여 customer별 transaction 체인을 만든다.

- 각 거래마다 Transaction node(date, price, salesChannel) 생성 후 `(t)-[:HAS_ARTICLE]->(a)` 연결.
- 첫 거래는 `(c)-[:START_TRANSACTION]->(t)`로, 최신 거래는 `(c)-[:LATEST]->(t)`로 추적. 새 거래가 오면 LATEST를 이동하고 이전 거래와 `[:NEXT]`로 연결.

```
(Customer)-[:START_TRANSACTION]->(T1)-[:NEXT]->(T2)-[:NEXT]->...->(Tn)<-[:LATEST]-(Customer)
   each Transaction -[:HAS_ARTICLE]-> (Article)
   (Customer)-[:LIVES_IN]->(PostalCode)
   (Article) -> ProductType / ColorGroup / Department / Section ... (fanned-out nodes)
```

- 최종 graph(Figure 8.1): article 속성이 개별 node로 fan-out, Customer는 우편번호·첫/마지막 transaction에 연결, Transaction은 Article 및 다음 transaction에 연결.

---

## 2. Optimizing for recommendations: best practices in graph modeling

> Neo4j는 schema optional이므로 후처리로 추가 relationship을 만들어 season·year 등 새로운 방식으로 데이터를 소비할 수 있다.

- Seasonal relationship 생성: 각 customer의 transaction을 START_TRANSACTION→LATEST 경로로 순회하며 month·year로 season(WINTER/SPRING/SUMMER/FALL) 판별.
- 12월은 해당 year, 1~2월은 year-1로 처리하여 `season+'_'+year`(예: WINTER_2019) relationship 이름 생성.
- 각 season의 첫 transaction에 `apoc.create.relationship`으로 동적 이름의 relationship 생성(relationship 이름이 동적이라 apoc 사용).
- 강화 후 graph(Figure 8.2)는 데이터에 대한 이해를 추가 맥락으로 담아 같은 데이터를 다르게 보고 지능을 추출하게 한다.

### Seasonal query 예시

- "특정 customer가 2019 여름에 산 article" 조회:
  - `MATCH (c:Customer)-[:SUMMER_2019]->(start), (c)-[:FALL_2019]->()<-[:NEXT]-(end)`로 여름·가을 모두 구매한 고객 선택.
  - SUMMER_2019를 시작점, FALL_2019 직전 transaction을 끝점으로 NEXT 순회하며 HAS_ARTICLE로 article 조회.
- property filter 대신 graph traversal에 전적으로 의존하여 매우 효율적으로 실행되며 query가 읽기 쉽고 설명 가능하다.
- 수동 로드를 원치 않으면 제공된 database snapshot(.zip dump)을 다운로드 가능.

---

## Summary (핵심 정리)

- 데이터를 소비하는 방식에 맞춰 모델링하면 효율적 검색이 가능하며, H&M 데이터셋을 이 원칙(label·정규화·transaction train)으로 로드했다.
- seasonal relationship 등 후처리로 graph에 맥락을 추가하여 query를 더 읽기 쉽고 설명 가능하게 만들었다.
- 다음 Chapter 9에서는 LLM으로 이 데이터를 더 강화하여 더 강력한 knowledge graph를 만드는 방법을 다룬다.
