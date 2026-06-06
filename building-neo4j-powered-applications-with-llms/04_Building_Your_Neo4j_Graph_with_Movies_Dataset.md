# 04. Building Your Neo4j Graph with Movies Dataset

## 챕터 개요 (3줄 요약)

- 효율적 검색을 위한 Neo4j graph 설계 원칙(node/relationship 타입, indexing, constraints)을 다룬다.
- TMDb(The Movie Database) 데이터셋을 정규화·정제하여 90K+ node, 320K+ relationship의 movie knowledge graph를 구축한다.
- variable-length relationship, path pattern, subquery, CALL IN TRANSACTIONS 등 advanced Cypher 기법을 소개한다.

---

## 1. Design considerations for a Neo4j graph for an efficient search

> 데이터가 graph에 조직되는 방식이 검색 성능과 결과 relevance에 직접 영향을 미치므로 효과적인 graph 모델링 원칙 이해가 중요하다.

### Node와 relationship 타입 정의 고려사항

- 영화 데이터에서 전통적으로 Movies, Actors, Directors, Genres를 별도 node로 두지만, 유사 entity를 단일 node로 통합하는 것이 더 효율적이다.
- Actor/Director를 분리하지 않고 단일 Person node로 만들고, Movie와의 relationship 타입(ACTED_IN, DIRECTED)으로 역할을 구분한다.
- 장점: 단순화된 데이터 모델, query 성능 향상(node 타입이 적을수록 traverse 효율↑), redundancy 감소, 유연한 relationship 정의, 유지보수·확장성 용이.

### Indexing과 constraints

> index는 query 시작점을 빠르게 찾게 해 검색 성능을 크게 높이고, constraint는 중복 node·잘못된 relationship을 방지해 데이터 무결성을 보장한다.

- person_name에 unique constraint를 걸어 단일 node로 표현하되, 동명이인 가능성이 있는 실제 환경에서는 person_id(IMDb/TMDb) 같은 신뢰할 수 있는 식별자를 사용한다.
- uniqueness를 강제하지 않으면 person_name에 index를, Movie.title에도 index를 생성해 빠른 조회를 지원한다.
- ACTED_IN relationship의 role property에 index 가능(Neo4j 5.x 이상에서 relationship property index 지원).

```
CREATE CONSTRAINT unique_person_id IF NOT EXISTS FOR (p:Person) REQUIRE p.person_id IS UNIQUE;
CREATE INDEX movie_title_index IF NOT EXISTS FOR (m:Movie) ON (m.title);
CREATE INDEX acted_in_role_index IF NOT EXISTS FOR ()-[r:ACTED_IN]-() ON (r.role);
```

---

## 2. Utilizing a movies dataset

> TMDb(Kaggle)는 45,000+ 영화의 제목·장르·cast·crew·평점 등 메타데이터를 담아 영화 산업의 복잡한 관계를 graph로 구축할 견고한 기반을 제공한다.

### 정규화·정제가 필요한 이유

- Consistency: 장르 형식 차이·중복 등을 일관된 형식으로 정규화. Neo4j는 Cypher pattern matching, APOC 병합 procedure, GDS node similarity로 entity linkage·deduplication을 지원한다.
- Efficiency: redundancy를 줄여 저장 요구량 최소화·query 성능 최적화.
- Accuracy: 부정확한 레코드 제거·수정으로 신뢰할 수 있는 insight 확보.
- Scalability: 표준화된 구조로 데이터 증가 시에도 관리·성능 유지.

### CSV 파일 정제·정규화

- credits.csv: stringified JSON에서 cast(actor_id, name, character)와 crew(Director, Producer만)를 추출·explode·normalize하여 normalized_cast.csv, normalized_crew.csv 생성.
- keywords.csv: 영화 plot keyword를 추출해 tmdbId별로 집계.
- movies_metadata.csv: 45,000 영화의 budget·revenue·release date 등을 genres, production companies, countries, spoken languages 노드별 CSV로 분리. belongs_to_collection에서 collection name 추출.
- links.csv / links_small.csv: 외부 DB 연결용이나 본 분석에서는 미사용.
- ratings.csv(2,600만 평점)는 미사용, ratings_small.csv(700 user, 9,000 영화의 10만 평점)를 사용.

---

## 3. Building your movie knowledge graph with code examples

> 정규화된 CSV를 AuraDB Free 인스턴스에 import하여 완전한 knowledge graph로 변환한다.

### AuraDB Free 인스턴스 설정

- console.neo4j.io 접속 → 로그인 → Create Free Instance.
- 연결 정보(NEO4J_URI, NEO4J_USERNAME, NEO4J_PASSWORD, AURA_INSTANCEID 등)를 안전하게 저장.

### 데이터 import (graph_build.py)

- public cloud storage(storage.googleapis.com/movies-packt/...)에서 CSV를 fetch하므로 수동 업로드 불필요.
- 먼저 unique constraint(tmdbId, movieId, company_id, genre_id 등)와 index(actor_id, crew_id, user_id 등) 생성.
- load_movies(), load_genres(), load_production_companies(), load_person_actors(), load_person_crew() 등으로 node 로드.
- relationship 생성: HAS_GENRE(Movie→Genre), PRODUCED_BY(Movie→ProductionCompany), HAS_LANGUAGE, PRODUCED_IN, ACTED_IN, DIRECTED, PRODUCED, RATED(Movie→User).
- `python graph_build.py` 실행 → .env 자격증명으로 연결 → DB 정리 → index/constraint 추가 → node·relationship 대량 로드.

```
(Movie)-[:HAS_GENRE]->(Genre)
(Movie)-[:PRODUCED_BY]->(ProductionCompany)
(Person)-[:ACTED_IN]->(Movie)
(Person)-[:DIRECTED]->(Movie)
(Person {role:'user'})-[:RATED {rating}]->(Movie)
```

- 결과: 90K+ node, 320K+ relationship의 연결된 movie graph(Figure 4.1).

---

## 4. Beyond the basics: advanced Cypher techniques

> graph가 커질수록 복잡한 구조를 다루기 위해 Cypher의 고급 기능이 필요하다.

- Variable-length relationships: `-[:ACTED_IN*1..3]-`로 1~3 단계 경로 매칭. social network 분석·계층 데이터 탐색에 적합.
- Pattern matching with path patterns: `MATCH path = (a:Actor)-[:ACTED_IN]->(m:Movie)`로 named path 정의·재사용. path chaining으로 배우-감독 반복 협업 같은 간접 관계 발견.
- Subqueries(CALL {...}): 복잡한 query를 모듈화. 예) action 영화 subquery 후 외부 query에서 감독 매칭.
- Procedural logic(CALL IN TRANSACTIONS): load_ratings()에서 `CALL (row) {...} IN TRANSACTIONS OF 50000 ROWS`로 대용량 CSV를 청크 단위로 로드하며 성능·트랜잭션 무결성 유지.
- Nested queries: revenue > 1억 영화를 필터링 후 연관 장르를 찾는 등 다중 기준 정제.

---

## Summary (핵심 정리)

- raw 데이터를 정제·정규화하고, Person node 통합 등 검색 효율을 높이는 graph 모델링 best practice를 적용했다.
- TMDb 데이터로 90K+ node·320K+ relationship의 movie knowledge graph를 AuraDB에 구축하고 advanced Cypher 기법을 익혔다.
- 다음 Chapter 5에서는 Haystack을 Neo4j에 통합하여 강력한 검색 기능을 구축하는 방법으로 이어진다.
