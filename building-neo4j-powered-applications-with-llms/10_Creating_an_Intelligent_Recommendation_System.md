# 10. Creating an Intelligent Recommendation System

## 챕터 개요 (3줄 요약)

- Neo4j Graph Data Science(GDS) 알고리즘과 머신러닝으로 추천을 한 단계 더 개선한다.
- KNN(K-Nearest Neighbors)으로 고객 유사도를, Louvain으로 community detection을 수행해 고객을 그룹화한다.
- collaborative filtering과 content-based 접근을 결합하여 커뮤니티·article 특성 기반 추천을 제공한다.

---

## 1. Improving recommendations with GDS algorithms

> GDS 알고리즘으로 graph를 강화하여 더 나은 추천 시스템을 구축한다. 환경: Neo4j Desktop, APOC 5.21.2, GDS 2.9.0.

- 절차: ① embedding 기반 KNN으로 고객 간 similar relationship 생성 → ② Louvain community detection으로 유사 관계 기반 고객 그룹화.
- similarity·community 알고리즘은 시간이 오래 걸려 사전 처리된 database dump(SUMMER_2019_SIMILAR·community 포함) 사용 권장.

### Computing similarity with KNN

> KNN은 node 쌍의 거리를 계산해 각 node와 top K 이웃 간 relationship을 생성하며, homogeneous graph가 필요하다.

- 3가지 모드: Stream(결과 스트리밍·검사), Mutate(in-memory graph만 갱신), Write(실제 DB에 결과 기록).
- Graph projection: SUMMER_2019 relationship의 embedding을 source node property로 사용해 in-memory graph 생성.
- `gds.knn.write`로 cosine similarity 계산, similarityCutoff 0.9, topK 5로 SUMMER_2019_SIMILAR relationship과 score 기록.
- cosine similarity: 0에 가까우면 비유사, 1에 가까우면 유사. 요약 텍스트 기반 embedding이라 0.9 cutoff로 유사도 과대평가 방지.
- KNN은 기본 non-deterministic(결정적 결과는 concurrency=1, randomSeed 명시 필요). 실행 후 `gds.graph.drop`으로 projection 메모리 해제.

### Detecting communities with Louvain

> Louvain은 entity 간 유사도 점수로 커뮤니티를 형성하며, modularity score를 최대화하는 계층적 clustering 알고리즘이다.

- Graph projection: SUMMER_2019_SIMILAR relationship과 score로 communityGraph 생성(undirectedRelationshipTypes ['*']).
- `gds.louvain.write`로 community ID를 Customer node의 summer_2019_community property에 기록(communityCount, modularity 반환).
- 커뮤니티별 고객 수 조회 가능(Figure 10.2). Louvain도 기본 non-deterministic. 처리 후 projection drop.

```
[Customer embeddings on SUMMER_2019 rel]
   --KNN(cosine, topK=5, cutoff=0.9)--> SUMMER_2019_SIMILAR relationships
   --Louvain--> summer_2019_community (Customer node property)
```

---

## 2. Understanding the power of communities

> 단순 vector similarity로 유사 고객을 찾는 것보다 커뮤니티가 왜 더 나은지 살펴본다.

- community 133(약 1,242명): 구매 요약에서 casual+lingerie를 함께 사는 경향이 드러나고, 실제 구매 article(dress+non-wired bra 등)이 요약과 잘 일치.
- 연령대 상관: 대부분 커뮤니티가 Youth(20~30세) 우세(Figure 10.3). community 5823은 Adult 우세.
- community 5823 요약: kids 의류·swimwear 등 "아이가 있는 고객" 경향 — vector similarity만으로는 놓칠 수 있는 관련 추천을 커뮤니티가 포착.
- vector similarity는 개별 유사 고객만 반환해 이질성이 있고, 잠재적으로 관련 있는 추천을 놓칠 수 있다. 커뮤니티는 구매 행동을 더 잘 이해하게 한다.

---

## 3. Combining collaborative filtering and content-based approaches

> collaborative filtering(고객/article 유사도 기반)과 content-based filtering(article 속성 기반)을 결합해 더 나은 추천을 제공한다.

### Scenario 1: 다른 커뮤니티 article 필터링

- 대상: community 1696의 특정 고객.
- 절차: ① 고객이 산 article 수집 → ② 같은 커뮤니티 고객이 산 article 수집 → ③ 다른 커뮤니티 고객이 산 article 수집 → ④ `apoc.coll.subtract`로 다른 커뮤니티 article 제거(커뮤니티 고유 article만) → ⑤ 고객이 이미 산 article 제거 → 10개 추천.
- 결과(Figure 10.4): jeans, jumper, pyjamas 등 고객 구매 요약에 부합하는 추천.

### Scenario 2: article 특성(section) + 다른 커뮤니티 필터링

- 대상: community 5823의 고객(주로 "Kids Boy" section 구매).
- 절차: Scenario 1과 동일하되 `(a)-[:HAS_SECTION]->(s)`로 "Kids Boy" section에 속하는 article만 대상으로 필터링.
- 같은 커뮤니티 고객의 Kids Boy article에서 다른 커뮤니티 article과 고객 기구매 article을 제거하여 10개 추천.
- 결과(Figure 10.5): kids용 jeans·shorts·top 등 구매와 article 속성을 모두 고려한 추천.

```
추천 = (같은 커뮤니티 article) - (다른 커뮤니티 article) - (고객 기구매 article)
       [+ 선택적: HAS_SECTION 등 article 특성 필터]
```

---

## Summary (핵심 정리)

- KNN similarity와 Louvain community detection으로 graph를 강화하여 데이터의 숨은 insight를 발굴했다.
- 커뮤니티 기반 collaborative filtering과 article 속성 기반 content-based 접근을 결합해 더 적절한 추천을 제공했다.
- 다음 장에서는 이러한 애플리케이션을 클라우드에 배포하는 방법과 배포 best practice를 다룬다.
