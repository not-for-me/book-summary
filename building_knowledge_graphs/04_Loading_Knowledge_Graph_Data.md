# 4. Loading Knowledge Graph Data

## 챕터 개요 (3줄 요약)
- 지식그래프 구축의 출발점은 기존 시스템 데이터의 대량 적재(bulk import)이며, Neo4j는 Data Importer / LOAD CSV / neo4j-admin import 세 가지 방식을 제공한다.
- 핵심 실무 원칙은 "작게 시작하라(start small)" — 대표 샘플로 모델을 먼저 검증한 뒤 전체 적재를 실행하라.
- 온라인(라이브 DB) 적재와 오프라인(초고속·DB 중단) 적재의 트레이드오프를 이해하고, 라이프사이클에 따라 여러 도구를 혼용한다.

## Loading Data with the Neo4j Data Importer
> Data Importer는 도메인 모델을 시각적으로 그리고 CSV 데이터를 노드·관계에 매핑하는 GUI 도구로, 초보자·전문가 모두에게 유용하다.

- CSV 헤더 규약: 노드는 `:ID(Person),name`, 관계는 `:START_ID(Person),:END_ID(Place),since` 형식. ID는 그래프 연결용이며 도메인 모델의 일부가 아닐 수 있다.
- 생성된 Cypher(CREATE CONSTRAINT, UNWIND, MERGE, SET)를 보여줘 검증·디버깅 가능하며, 소스 관리에 넣어 진화시킬 수 있다.
- 모델과 데이터 흐름을 적재 전에 시각적으로 검증한다는 점이 가장 큰 가치.

## Online Bulk Data Loading with LOAD CSV
> LOAD CSV는 라이브 DB에 CSV(웹 URL, S3, Google Sheets, 파일시스템, 압축 파일 포함)를 적재하는 Cypher 명령이다.

- 기본형: `LOAD CSV WITH HEADERS FROM 'places.csv' AS line MERGE (:Place {...})`.
- 불규칙 데이터 처리: null 속성을 직접 MERGE 패턴에 넣으면 실패 → 관계/노드 생성 후 `SET`으로 속성 추가(트랜잭션 원자성 유지).
- 대량(100만건+) 적재는 배치 분할: Neo4j 4.4+ `CALL {...} IN TRANSACTIONS OF N ROWS` (이전엔 apoc.periodic.iterate).
- 일반 Cypher이므로 EXPLAIN/PROFILE로 튜닝 가능 — Cartesian product, eager operator(전체 데이터를 한꺼번에 당겨 병목) 주의.

```
LOAD CSV WITH HEADERS FROM "lives_in.csv" AS line
MATCH (person:Person {name:line.from}), (place:Place {city:line.to})
MERGE (person)-[r:LIVES_IN]->(place)
SET r.since = line.since   // null 안전
```

## Initial Bulk Load (neo4j-admin import)
> neo4j-admin import는 CSV로부터 새 DB를 만드는 오프라인 초고속 임포터로, 초당 약 100만 레코드 처리.

- 매우 빠른 대신 DB가 적재 중 오프라인(unavailable). 재개 불가(Ctrl-c 시 처음부터 다시).
- 파일시스템만 읽음(S3 불가), gzip 압축 지원. 노드/관계별로 파일 분리, 헤더 파일과 데이터 파일 분리(대용량 텍스트 편집 편의).
- "완벽할 필요 없고 충분히 좋으면 됨" — 구분자 차이·중복·불량 관계는 관용적으로 처리하나 특수문자·BOM 등은 주의.

## Summary (핵심 정리)
- 세 가지 적재 방식의 선택 기준: 시각 검증·학습 → Data Importer, 라이브 증분 적재 → LOAD CSV, 초기 대용량 부트스트랩 → neo4j-admin import(오프라인).
- 불규칙·결측 데이터는 MERGE 후 SET으로 안전하게 처리하고, 대량 적재는 배치로 분할해 DB 부하를 관리한다.

> [모델링 관점 - 주식시장 도메인 적용]
> 주식시장 데이터는 출처가 다양하고(거래소 시세, 공시/DART·EDGAR, 재무제표, 뉴스, 지분공시) 규모가 크며 결측이 흔하다. 따라서: (1) 초기 종목·기업·거래소 마스터는 neo4j-admin import로 부트스트랩, (2) 일별 가격·공시 같은 증분 데이터는 LOAD CSV + IN TRANSACTIONS 배치로 라이브 적재, (3) 종목 식별자(ISIN/ticker)를 ID 컬럼으로 삼아 서로 다른 소스를 안전하게 정합(MERGE+제약). "작게 시작" 원칙은 금융처럼 스키마가 복잡한 도메인에서 모델 검증 비용을 크게 줄여주므로 특히 중요하다. 결측 재무 항목은 SET 패턴으로 흡수해 현실 데이터의 불완전성을 수용한다.
