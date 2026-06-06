# 06. Exploring Advanced Knowledge Graph Capabilities with Neo4j

## 챕터 개요 (3줄 요약)

- 기본 검색을 넘어 multi-hop reasoning, dynamic filter, graph reasoning 등 고급 knowledge graph 탐색 기법을 다룬다.
- Neo4j와 Haystack 통합의 확장성(query 최적화, caching, vector indexing, 수평 확장)을 설명한다.
- AI 검색 시스템의 모니터링·알림·로깅·유지보수 best practice를 제시한다.

---

## 1. Exploring advanced Haystack functionalities for knowledge exploration

> 단순 similarity 매칭을 넘어 graph의 다층적 knowledge를 탐색하는 context-based reasoning과 use case별 검색 최적화를 다룬다.

### Context-aware search (multi-hop reasoning)

- Neo4j graph의 multi-hop reasoning과 Haystack similarity search를 결합해 node 간 path를 탐색하며 맥락을 더한다.
- 예: `MATCH (m:Movie {title})<-[:DIRECTED]-(d:Director)-[:DIRECTED]->(related:Movie)`로 Inception 감독의 다른 영화 검색 후 Haystack으로 랭킹.
- 데이터셋 일부만 import한 경우 one-to-many 관계가 없어 "No related movies found" 발생 가능(전체 데이터셋·AuraDB Professional 업그레이드 권장).

### Dynamic search queries with flexible filters

- knowledge graph의 강점은 검색 시 동적 filter 적용이다.
- Neo4jEmbeddingRetriever에 filter(예: `release_date >= "1995-11-17"`)를 적용해 결과를 정제.
- 카테고리·평점·시간 범위 등으로 가장 관련성 높은 결과로 좁힐 수 있어 상호작용적·맥락 풍부한 검색 시스템 구축에 핵심.

### Search optimization

- perform_optimized_search()에서 top_k 파라미터를 조정해 반환 결과 수를 fine-tune(모델 자체가 아닌 vector similarity 기반 결과 수).

---

## 2. Graph reasoning with Haystack

> 텍스트 유사성만 검색하는 전통적 방식과 달리, graph reasoning은 entity 간 풍부한 relationship을 활용해 더 깊은 insight를 발굴한다.

### 다중 relationship 순회로 숨은 insight 발굴

- 여러 relationship 타입과 multi-hop query로 직접 관계를 넘는 패턴 발견 후 Haystack similarity search로 정제·랭킹.
- 예: Jurassic Park와 같은 배우·감독을 공유하는 영화 검색 — `(m:Movie {title})<-[:ACTED_IN|DIRECTED]-(p)-[:ACTED_IN|DIRECTED]->(related:Movie)`에 CASE로 role(Actor/Director) 판별.

### Path query로 insight 발굴

- 두 영화가 일련의 협업으로 어떻게 연결되는지 path query로 탐색.
- 예: `MATCH path = (m1:Movie {title:"Inception"})-[:ACTED_IN*3]-(m2:Movie) RETURN m1.title, m2.title, path` — 공유 배우를 통한 3-hop 연결.

```
Inception --ACTED_IN--> Actor A --ACTED_IN--> Movie B --ACTED_IN--> Actor B --ACTED_IN--> Movie C
(three-hop undirected traversal; movie<->actor 전환 반복)
```

- multi-hop reasoning은 content discovery, recommendation system, collaboration network 분석에 유용하다.

---

## 3. Scaling your Haystack and Neo4j integration

> 시스템이 확장되면 Haystack·Neo4j 모두에 부하가 증가하므로 query 최적화, caching, indexing, 수평 확장이 중요하다.

### 대규모 graph의 Neo4j query 최적화

- Index·constraint 사용: title, name 등 자주 조회하는 property에 index 생성으로 node lookup·traversal 가속.
- Profile·Optimize: PROFILE/EXPLAIN 키워드로 query 성능 분석, 병목 파악.
- Limit Results Early: 큰 결과셋은 query 초반에 LIMIT으로 over-fetching 방지.

### Caching (embedding·query 결과)

- Embedding caching: 자주 묻는 query의 embedding을 Neo4j나 Redis 등 캐시 레이어에 저장해 재계산 방지.
- Query 결과 caching: Redis·Memcached로 인기 query 결과를 캐싱해 Neo4j 부하 감소.

### Vector indexing 효율화

- vector index를 embedding 차원(1536)·cosine에 맞게 최적 구성.
- Batch write: 많은 embedding 작성 시 batch_size=100 등 batch 연산으로 개별 write 오버헤드 감소.

### Load balancing과 수평 확장

- Scale Neo4j: AuraDB나 Neo4j cluster로 워크로드를 여러 인스턴스에 분산.
- Load balance Haystack: 여러 Haystack 인스턴스에 query 분산으로 일관된 성능·고가용성 유지.
- Kubernetes: 컨테이너화된 Haystack을 replicas(예: 3)로 배포하여 트래픽에 따라 동적 확장.

---

## 4. Best practices for maintaining and monitoring your AI-powered search system

> 초기 구축 이후 장기적 성공을 위해 성능 점검·능동적 모니터링·견고한 로깅 전략이 필수다.

- Neo4j·Haystack 모니터링: Prometheus·Grafana로 query 응답시간·throughput·latency·시스템 부하 시각화.
- 알림 설정: Prometheus Alertmanager나 Grafana로 느린 query·검색 실패·부하 증가에 임계값 기반 알림.
- 로깅 전략: query 실행시간·실패·리소스 사용을 Haystack·Neo4j 양쪽에 기록해 root cause 분석.
- 정기 유지보수: Neo4j(index 재구축, 데이터 일관성 점검, 디스크 모니터링), Haystack(embedding 품질 모니터링, 모델 업데이트, document store 증가 관리).

---

## Summary (핵심 정리)

- multi-hop reasoning, dynamic filtering, path query 등으로 graph reasoning을 수행하여 숨은 연결과 깊은 insight를 발굴했다.
- caching, 효율적 indexing, query 최적화, 수평 확장으로 Haystack·Neo4j 통합의 확장성을 확보했다.
- 다음 Part 3에서는 Spring AI·LangChain4j와 Neo4j를 통합해 정교한 recommendation system을 구축하는 방향으로 전환한다.
