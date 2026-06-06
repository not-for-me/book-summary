# 5. Integrating Knowledge Graphs with Information Systems

## 챕터 개요 (3줄 요약)
- 지식그래프의 진짜 힘은 다른 시스템과 통합되어 서로를 풍부하게 하는 "선순환(virtuous cycle)"에서 나오며, 그래프가 데이터 패브릭(Data Fabric)의 중심 인덱스 역할을 한다.
- 통합 수단은 다층적이다: 드라이버, 그래프 연합(Composite DB), 서버사이드 프로시저, 데이터 가상화, 커스텀 함수, 그리고 GraphQL·Kafka·Spark·ETL 같은 보완 도구.
- 핵심 원칙은 "복사하지 말고 가상화·인덱싱하라" — 원본을 옮기지 않고 지식그래프를 통해 통합된 골든 레코드 뷰를 제공한다(단, 원격 호출 지연시간 고려).

## Towards a Data Fabric
> 데이터 패브릭은 조직 전체의 데이터 접근 계층으로, 지식그래프가 silo를 가로지르는 큐레이션된 인덱스(골든 레코드)를 제공한다.

- 클라이언트는 백엔드(문서DB/관계형/그래프)의 물리적 위치를 모른 채 "John Smith from Seattle"을 그래프 관계를 따라 조회.
- 그래프의 유연성으로 동일 엔터티의 이질적 표현을 점진적으로 연결 — big up-front 설계 불필요.
- 조직 원리(온톨로지/택소노미)를 얹어 시스템 간 데이터 검증·일관성 확인. 노드 차수·이웃·중심성 메트릭을 매칭 규칙(예: 이름 유사도 95%+ 및 동일 군집계수 → 중복 제거)에 활용.

## The Database Driver
> 드라이버는 앱과 그래프 DB를 네트워크로 잇는 클라이언트사이드 미들웨어(Java/.NET/JS/Python/Go 등)이다.

- 4대 원칙: (1) Driver 객체는 비싸다 → 앱 수명 동안 하나만, (2) Session은 싸다 → 필요할 때마다, (3) 읽기/쓰기 여부를 알려 클러스터 라우팅 최적화, (4) 모든 쿼리를 파라미터화(재파싱 방지 + 인젝션 방어).

## Graph Federation / Server-Side Procedures / Data Virtualization
> Composite Database(연합)와 APOC 가상화는 데이터를 물리적으로 합치지 않고 단일 가상 그래프로 제공한다.

- Composite DB: `CREATE COMPOSITE DATABASE` + ALIAS로 EMEA/APAC 등 여러 물리 DB를 한 Cypher 쿼리로 페더레이션(`USE graph.byName(g)`).
- 서버사이드: APOC로 쿼리 시점에 SQL DB(apoc.load.jdbc)/MongoDB/JSON Web API를 호출해 복사 없이 그래프 보강.
- 데이터 가상화: `apoc.dv.catalog.add`로 외부 소스(시계열·로그)를 가상 노드로 매핑 → 통신망 디지털 트윈처럼 위상(그래프)+실시간 메트릭(외부 DB) 결합.

## Complementary Tools (GraphQL / Kafka / Spark / ETL)
> 주변 시스템에 따라 GraphQL(API), Kafka(스트리밍), Spark(대량 처리), ETL(Apache Hop 등)로 통합한다.

- GraphQL: Neo4j 구현의 `@relationship` 디렉티브로 타입 시스템에 관계를 표현, 사용자 근처 API 계층에 적합.
- Kafka Connect: 그래프를 소스(CDC) 또는 싱크로 — 주기적 Cypher 쿼리로 발행하거나 토픽 메시지를 MERGE로 적재.
- Spark Connector: 그래프를 소스/싱크로 읽고 씀(레이블·predicate·Cypher). SaveMode.ErrorIfExists→CREATE, Overwrite→MERGE. 표 모델↔그래프 모델 매핑 필요.
- ETL(Apache Hop 등): low-code로 소스↔그래프 매핑, 파라미터화 Cypher로 정교한 ingress/egress.

> [모델링 관점 - 주식시장 도메인 적용]
> 주식시장 데이터 플랫폼에서 지식그래프는 "데이터 패브릭의 의미 계층"으로 두는 것이 정석이다. 시세(시계열DB), 공시(문서DB), 재무(관계형), 뉴스(검색엔진)는 각자 최적 저장소에 두고, 그래프는 ticker/ISIN 골든 레코드로 이들을 연결하는 인덱스 역할을 한다. 모델링 시: (1) 고빈도 가격 시계열은 그래프에 적재하지 말고 APOC 데이터 가상화로 "필요할 때" 가상 노드로 끌어오기(지연시간 감수). (2) 실시간 공시/체결은 Kafka 싱크로 MERGE 적재해 그래프를 항상 최신으로. (3) 정량 분석·팩터 계산은 Spark로 그래프에서 읽어 처리 후 결과를 다시 그래프에 기록. (4) 프론트 대시보드는 GraphQL @relationship으로 "기업-지분-섹터" 그래프를 안전한 타입 계약으로 노출. 이 구조가 금융 도메인의 이질적 대규모 데이터를 "복사 없이" 통합하는 핵심이다.

## Summary (핵심 정리)
- 지식그래프는 데이터 패브릭의 중심에서 silo를 가로지르는 골든 레코드 인덱스를 제공하며, 통합은 드라이버·연합·가상화·스트리밍·ETL 등 다층적으로 선택한다.
- "복사하지 않고 가상화/인덱싱"이 핵심 철학이나, 원격 호출은 지연시간·신뢰성 비용을 수반하므로 고빈도 데이터는 신중히 다뤄야 한다.
- 주식시장 플랫폼에서는 그래프를 의미 계층으로 두고 시세/공시/재무/뉴스를 식별자로 연결하며, Kafka·Spark·GraphQL을 역할별로 배치하는 것이 실무 아키텍처의 정석이다.
