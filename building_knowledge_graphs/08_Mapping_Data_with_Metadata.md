# 8. Mapping Data with Metadata Knowledge Graphs

## 챕터 개요 (3줄 요약)
- 메타데이터 지식그래프(Metadata Knowledge Graph)는 데이터의 형태·위치, 처리 시스템, 소비자를 기록하는 "기업 전체의 데이터 지도"로, 데이터 거버넌스와 셀프서브 소비의 기반이다.
- 데이터·프로세스·소비자를 연결해 데이터 출처(provenance)와 계보(lineage)를 명시적으로 추론 가능하게 하며, 컴플라이언스·규제 대응에 핵심이다.
- 핵심 모델은 Dataset–DataPlatform–Field–Task/Pipeline–DataSink로 구성되며, 데이터 평면과 메타데이터 평면을 한 그래프에 결합하는 것이 강력하다.

## The Challenge of Distributed Data Stewardship
> 기업 데이터는 silo에 분산되고 이질적이며 품질이 들쭉날쭉해, "데이터가 어디 있고 누가 쓰는지" 파악 자체가 어렵다.

- 부서별 시스템이 전사 지식조직 관점 없이 구축되어 중복·유사 중복 데이터가 흩어짐.
- 메타데이터(데이터에 대한 데이터)는 전체 데이터 생태계를 보는 렌즈 — 분석가·데이터과학자·ML 엔지니어가 올바른 데이터를 빠르게 찾고 감사·규제를 설명하게 함.
- 대형 조직 사례: Airbnb Dataportal(2017), Lyft Amundsen, LinkedIn DataHub 등 "지식그래프 기반 메타데이터 허브" 트렌드.

## Core Model: Datasets / Tasks & Pipelines / Data Sinks
> 메타데이터 그래프는 현대 데이터 스택의 공통 자산(데이터셋·태스크/파이프라인·데이터싱크)을 노드로 포착한다.

- Dataset: 테이블/문서/스트림 등 모든 데이터 집합. `source` 관계로 DataPlatform(예: BigQuery)에 연결, Field 노드로 공개 스키마 기술.
- Task: 데이터를 처리하는 작업(예: ETL 표준화). 연쇄(chain)되어 Pipeline/Flow를 이루며 실행 순서·의존성을 명시.
- DataSink: BI 시각화·ML 학습셋 같은 최종 소비 — 새 데이터셋을 생성하지 않음. 모든 요소는 Domain·Owner·CatalogTerm(전사 온톨로지/용어집)에 연결 가능.

```
(Field {address}) -[:associated_with]-> (CatalogTerm {location_info})
(Dataset) -[:source]-> (DataPlatform {BigQuery})
(Task) -[:produces]-> (Dataset) -[:consumes]- (DataSink {Dashboard})
(Dataset) -[:owned_by]-> (User {role:'data steward'})
```

## Querying the Metadata Graph
> 메타데이터 그래프는 데이터 발견, 영향 분석(impact), 데이터 계보(lineage) 질의를 지원한다.

- 인기도: 데이터셋을 소비하는 DataSink 수로 측정(`count{ (d)<-[:consumes]-(:DataSink) }`).
- 영향 분석: 특정 Task 실패 시 영향받는 소비자·소유자 목록 — `(t:Task)-[:produces|consumes*2..]-(:Dataset)<-[:consumes]-(s:DataSink)`.
- 데이터 계보(역방향): "Dashboard X의 원천 플랫폼은?" — 소비에서 원천 DataPlatform까지 역추적.
- 그래프 알고리즘으로 팀·도메인 간 강한 연결/단절 분석 가능(데이터 흐름 지도 = 회사 운영의 프록시).

## Using Relationships to Connect Data and Metadata
> 같은 그래프에 데이터 평면(고객·구독)과 메타데이터 평면(출처·스튜어드)을 2계층으로 결합한다.

- "서비스 X 구독 고객은?"에 답하면서 동시에 출처·거버넌스 메타데이터로 신뢰성 제공.
- 소스 시스템에 비침투적(noninvasive)으로 그 위에 계층으로 구축 가능 — 분산 고객 뷰를 연결해 360도 고객 뷰·중복 제거·시맨틱 검색 기반 마련.

> [모델링 관점 - 주식시장 도메인 적용]
> 주식시장 데이터 플랫폼 구축에서 이 장은 "내가 만드는 플랫폼 자체의 메타데이터 모델"로 직접 적용된다. 시세·공시·재무·뉴스·팩터가 여러 소스/파이프라인을 거치므로: (1) 각 데이터셋(일별 OHLCV, 분기 재무, 공시 원문)을 Dataset 노드로, 처리 단계(정규화·복원수정·팩터 계산)를 Task 체인으로 모델링해 "이 팩터 값이 어느 원천에서 어떤 변환을 거쳤는지"의 lineage를 확보. 이는 금융 규제(MiFID/감사) 대응과 백테스트 재현성에 필수. (2) Field를 CatalogTerm(FIBO 용어)에 연결해 "address=location_info"처럼 컬럼 의미를 표준화. (3) 영향 분석 쿼리로 "특정 시세 피드 장애 시 영향받는 리스크 대시보드/모델"을 즉시 파악. 즉, 비즈니스 지식그래프(기업-지분-섹터)와 별개로, 그 그래프를 떠받치는 "데이터 거버넌스 메타그래프"를 함께 설계하는 것이 신뢰할 수 있는 인사이트의 토대다.

## Summary (핵심 정리)
- 메타데이터 지식그래프는 전사 데이터 자산·프로세스·소비자를 추적하는 기반 계층으로, 데이터 발견·영향 분석·계보 추적을 단일 그래프 질의로 가능케 한다.
- 데이터 평면과 메타데이터 평면을 결합하면 답과 함께 출처·거버넌스 근거를 제공하며, 소스 시스템에 비침투적으로 구축할 수 있다.
- 주식시장 플랫폼에서는 비즈니스 그래프와 별도로 데이터 계보·거버넌스 메타그래프를 설계해야 규제 대응·백테스트 재현성·신뢰성을 확보할 수 있다.
