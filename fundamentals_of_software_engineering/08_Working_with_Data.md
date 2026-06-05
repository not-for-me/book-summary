# 08. Working with Data

## 챕터 개요 (3줄 요약)
- 데이터는 거의 모든 애플리케이션의 근간이며, 데이터 타입·저장·쿼리·마이그레이션을 다루는 능력은 코딩 능력만큼 중요하다.
- 구조적/비구조적 데이터와 다양한 포맷(JSON, XML, CSV, YAML)을 이해하고, 용도에 맞는 데이터베이스(관계형/문서/키-값/그래프/벡터)를 선택한다.
- 영속성 패턴·연결 풀·트랜잭션·일관성 모델·캐싱·쿼리 최적화·마이그레이션을 통해 성능과 신뢰성을 확보한다.

---

## 1. Understanding Data Types and Formats
> 소프트웨어는 데이터를 처리·변환·표현하는 시스템이며, 데이터 구조화·검증·처리 방식이 프로젝트 성공을 좌우한다.

- 데이터 작업 시 고려 요소: 청중(사람/기계), 성능 요구, 호환성, 복잡도, 검증 필요성.

### Structured vs Unstructured Data
- 구조적 데이터는 미리 정의된 스키마를 따라 행/열로 정리되어 쿼리·저장이 쉽다(예: Customer 클래스).
- 특징: 일관된 형식, 명확한 관계, 쉬운 쿼리, 효율적 저장 — 다만 유연성이 떨어진다.
- 비구조적 데이터는 사전 정의 모델이 없다(이메일, 이미지, 로그) — NLP(Natural Language Processing)나 컴퓨터 비전이 필요하다.
- 비구조적 데이터는 Elasticsearch 같은 전문 검색 엔진이나 데이터 레이크가 필요하다.

### Common Data Formats
- JSON(JavaScript Object Notation): 가독성·단순성으로 데이터 교환의 사실상 표준, 웹 API에 적합.
- XML(eXtensible Markup Language): 장황하지만 XSD/DTD로 강력한 검증, 네임스페이스 지원, 엔터프라이즈에 여전히 우세.
- CSV(Comma-Separated Values): 표 형식 평문, 단순하지만 타입 정의·특수문자 처리 규칙이 없음.
- YAML(YAML Ain't Markup Language): 사람 친화적, 주석 지원, 들여쓰기로 구조 정의 — DevOps 설정에 이상적.

### Specialized Data Considerations
- 이진 데이터(binary): 이미지/문서 등, Base64 인코딩으로 텍스트 포맷에 포함, 메모리 관리 주의.
- 날짜/시간: 시간대·DST·달력 체계로 복잡 — 내부적으로 UTC 저장, 교환은 ISO 8601, 최신 라이브러리 활용.
- 대용량 데이터셋: 스트리밍 처리, 페이지네이션, 인덱싱, 동시성 제어로 메모리 부담 회피.

---

## 2. Storing Your Data Effectively
> 적절한 저장 메커니즘 선택은 성능·확장성·유지보수성에 영향을 주며, 데이터 모양과 요구사항 이해가 핵심이다.

### Database Types and Their Use Cases
- 선택 요소: 데이터 구조 복잡도, 읽기/쓰기 패턴, 쿼리 복잡도, 확장 요구, 일관성 요구.

### Relational databases
- 데이터를 테이블의 행/열로 정리하고 키로 관계를 맺으며 ACID 속성으로 무결성을 보장한다.
- ACID: 원자성(Atomicity), 일관성(Consistency), 격리성(Isolation), 지속성(Durability).
- 명확한 관계, 복잡한 쿼리/조인, 트랜잭션이 중요할 때 사용(PostgreSQL, MySQL, Oracle).

### Document / Key-value / Graph / Vector databases
- 문서 DB: JSON 유사 문서로 유연한 스키마(MongoDB, Firestore) — 가변 구조, 수평 확장에 적합.
- 키-값 저장소: 가장 단순한 NoSQL, 초고속(Redis, DynamoDB) — 캐싱·세션 저장에 적합.
- 그래프 DB: 노드/엣지로 고도로 연결된 데이터(Neo4j) — 소셜 네트워크·추천 엔진에 적합.
- 벡터 DB: 고차원 임베딩 저장과 유사도 검색(Pinecone, Weaviate) — 시맨틱 검색·RAG(Retrieval-Augmented Generation)·LLM(Large Language Model)에 사용.

```
Database selection cheat sheet:
  Relational -> structured, ACID, complex joins
  Document   -> flexible/evolving schema
  Key-Value  -> blazing-fast simple access (cache/session)
  Graph      -> highly interconnected relationships
  Vector     -> AI embeddings + similarity search
```

---

## 3. Data Persistence and Management
> 언어/프레임워크마다 추상화 수준이 다르며, 애플리케이션 복잡도와 팀 역량에 맞는 수준을 선택한다.

- 직접 DB 접근(direct access): 원시 SQL, 완전한 제어이나 깊은 DB 지식 필요.
- 리포지토리 패턴(repository pattern): 비즈니스 로직과 데이터 접근 사이 추상화 — 관심사 분리, 테스트 용이.
- ORM(Object Relational Mapping): Hibernate/JPA 등 최고 추상화, 객체를 테이블에 매핑 — 높은 추상화가 항상 좋은 건 아니다.

### Connections & Transactions
- 대부분의 앱은 연결 풀(connection pool, 예: HikariCP)로 연결을 재사용한다.
- 트랜잭션은 여러 작업을 단일 원자적 단위로 실행해 완전 성공 또는 완전 실패를 보장한다(예: 계좌 이체).

---

## 4. Consistency Models and Caching Strategies
> 일관성 모델은 시스템 전반의 데이터 정확성 처리 방식을 정의하고, 캐싱은 응답 시간과 DB 부하를 줄인다.

### Consistency Models & CAP
- 강한 일관성(strong): 모든 부분이 동시에 같은 데이터를 봄(정확성 우선).
- 최종 일관성(eventual): 일시적 불일치 허용, 가용성 우선(DynamoDB).
- 인과적(causal)/세션(session) 일관성은 그 중간 지점을 제공한다.
- CAP 정리: 분산 시스템에서 일관성·가용성·분할 내성 중 둘만 보장 가능 — CP, AP, CA 시스템.

### Caching Strategies
- Cache-aside(lazy loading): 앱이 캐시/DB를 직접 관리, 읽기 많은 앱에 적합.
- Write-through: 쓰기 시 캐시/DB 동시 갱신, 강한 일관성이나 쓰기 느림.
- Write-behind(write-back): 캐시 즉시 갱신 후 DB 비동기 갱신, 빠르지만 데이터 손실 위험.
- 캐싱 판단: 데이터 변경 빈도, 오래된 데이터 비용, 쿼리 비용, 병목 여부를 자문한다.

---

## 5. Querying and Managing Data Performance
> 쿼리와 데이터 관리 방식이 애플리케이션 성능을 좌우하며, 작은 최적화가 수초를 수밀리초로 줄일 수 있다.

### Efficient Query Writing
- SELECT *를 피하고 필요한 컬럼만 선택하며 LIMIT/페이지네이션으로 결과 집합을 제한한다.
- 준비된 문(prepared statement)은 쿼리 플랜을 캐싱해 20~50% 단축하고 SQL 인젝션을 방지한다.
- 인덱스(index)는 전체 테이블 스캔을 빠른 조회로 바꾸지만(예: 2000ms→5ms), 저장 공간과 쓰기 속도를 희생한다.
- WHERE/JOIN/ORDER BY/GROUP BY에 자주 쓰는 컬럼을 인덱싱하고, 카디널리티 낮은 컬럼은 피한다.
- 대용량 결과는 오프셋 대신 키셋 페이지네이션(keyset pagination)을 고려한다.

### Tools and Best Practices
- 쿼리 플래너(planner/optimizer)는 SQL을 실행 계획으로 변환하며 대부분 비용 기반(cost-based)이다.
- EXPLAIN/EXPLAIN ANALYZE로 실행 계획을 확인하고 전체 테이블 스캔, 비효율 조인 등 경고 신호를 찾는다.
- 관찰성(observability)은 로깅·메트릭·추적으로 시스템 내부 상태를 파악한다(Spring Boot Actuator + Micrometer).
- N+1 쿼리 문제, 연결 풀 병목 등을 추적으로 가시화한다.
- 가독성·유지보수성과 쿼리 성능의 균형을 맞추되, 종종 ORM+원시 SQL 하이브리드가 최선이다.

---

## 6. Data Migration and Transformation
> 데이터 마이그레이션은 모든 엔지니어가 마주하는 도전이며, 신중한 계획·실행·검증이 필요하다.

### Movement Fundamentals
- 빅뱅(big bang) 마이그레이션은 다운타임에 모든 데이터를 한 번에 이동 — 쉽지만 위험이 크다.
- 단계적(phased) 마이그레이션은 검증하며 세그먼트별로 이동 — 느리지만 안전, 초보자에게 권장.
- ETL(Extract, Transform, Load): 추출→변환→적재의 3단계로 시스템 간 데이터를 이동한다.
- 동기화는 메시지 큐, CDC(Change Data Capture), 조정(reconciliation) 프로세스를 활용한다.

### Handling Schema Changes
- 스키마도 코드처럼 버전 관리해야 한다(Flyway, Liquibase, Rails Migrations).
- Flyway는 flyway_schema_history 테이블로 적용된 마이그레이션을 추적하고 순차적으로 실행한다.
- 마이그레이션 스크립트는 멱등(idempotent)하고 가능한 한 하위 호환되게 작성한다.
- 스키마 변경과 데이터 변환을 함께 버전 관리해 원자성을 유지하고, 작은 샘플로 먼저 검증한다.

---

## Summary (핵심 정리)
- 데이터는 대부분 애플리케이션의 근간이며, 효과적으로 다루는 능력이 성공을 좌우하는 가치 있는 기술이다.
- 데이터 타입/포맷 이해, 용도에 맞는 DB 선택, 영속성 패턴·연결 관리, 쿼리 최적화, 성장·마이그레이션 계획을 익힌다.
- 데이터 관리에 완벽한 해법은 없고 적절한 해법만 있으므로, 과잉 설계(overengineering)를 피하고 현재 요구에 맞춰 결정하고 나아간다.