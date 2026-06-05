# 02. Assembling the Building Blocks of a Reliable Data System

## 챕터 개요 (3줄 요약)

- 데이터 다운타임은 사후 해결보다 올바른 시스템·프로세스로 예방하는 것이 핵심이며, 본 장은 그 "기술(technology)" 구성요소를 다룬다.
- 운영(operational) vs 분석(analytical) 데이터, 데이터 웨어하우스 vs 레이크 vs 레이크하우스의 차이와 각 단계의 품질 관리법을 설명한다.
- 데이터 품질 메트릭 수집 방법(Snowflake 예시), 쿼리 로그 활용, 데이터 카탈로그/디스커버리 설계까지 신뢰 가능한 시스템의 메타데이터 기반 빌딩 블록을 정리한다.

---

## 1. Understanding the Difference Between Operational and Analytical Data (운영 데이터와 분석 데이터의 차이)

> 운영 데이터는 비즈니스를 "운영(run)"하고, 분석 데이터는 비즈니스를 "관리(manage)"한다. 둘의 구분은 데이터 품질 문화 도입의 출발점이다.

- 운영 데이터: 일상 운영에서 생성되는 데이터(재고 스냅샷, 고객 임프레션, 거래 기록 등).
- 분석 데이터: 데이터 기반 의사결정의 근거가 되는 데이터(마케팅 이탈률, 클릭률, 지역별 임프레션 등).
- 분석 데이터는 보통 운영 데이터의 변환·집계 위에 구축된다(운영이 분석의 상류/upstream).
- 이 구분은 OLTP(Online Transaction Processing) vs OLAP(Online Analytical Processing) 비교와 동일하다.
- 본 책은 분석 데이터의 품질에 초점을 두며, 운영 데이터 신뢰성은 주로 DevOps/SRE 영역이다.

### What Makes Them Different? (무엇이 다른가? — 처리량-지연 트레이드오프)
- 핵심은 throughput(단위 시간당 처리량) vs latency(처리 전 지연)의 트레이드오프이다.
- 고정된 연산 자원에서는 둘을 동시에 최적화할 수 없다(카페 비유: 계산대 집중→지연↓ 처리량↓).
- 운영 DB는 저지연(low latency)에, 분석 DB는 고처리량(high throughput)에 최적화된다.
- 따라서 고객 UI에서 Snowflake/Redshift를 직접 쿼리하거나, MySQL/Postgres에서 수조 행 집계를 돌리지 않는다.

---

## 2. Data Warehouses Versus Data Lakes (웨어하우스 vs 레이크)

> 웨어하우스는 "schema-on-write"로 구조화 데이터를, 레이크는 "schema-on-read"로 모든 포맷을 저장한다. 두 기술은 빠르게 수렴 중이다.

### Data Warehouses: Table Types at the Schema Level
- 데이터 진입 시점에 구조를 강제(schema-on-write)하여 데이터 위생을 강화하고 소비를 단순화한다.
- 1980년대 Kimball Group의 방법론에 기반하며, BI 도구(Looker, Tableau)와 즉시 연동된다.
- 대표 기술: Amazon Redshift(컬럼형, AWS), Google BigQuery(서버리스, GCP), Snowflake(컴퓨트·스토리지 분리 과금).
- 단점: 유연성 부족(테이블 형식 강제, JSON 등 반구조화 미지원), SQL-only(Python 미지원), 빠른 반복 작업엔 마찰적.

### Data Lakes: Manipulations at the File Level
- "schema-on-read"로 사용 시점에 구조를 추론하며, DIY 방식으로 메타데이터·스토리지·컴퓨트를 선택한다.
- James Dixon이 개념화했고 초기엔 Hadoop MapReduce/HDFS/Hive, 이후 Apache Spark로 발전.
- 특징: 스토리지-컴퓨트 분리, 분산 컴퓨팅, 커스터마이징/상호운용성, 오픈소스 기반, 비구조화 데이터 처리, 비 SQL 모델 지원.
- 도전과제: 데이터 무결성("blind ETL" 위험), Swampification(기술부채 누적으로 데이터 늪화), 더 많은 엔드포인트(오류 기회 증가).

### What About the Data Lakehouse?
- 웨어하우스와 레이크의 장점을 결합. 고성능 SQL(Presto/Spark), 스키마(Parquet), ACID(Delta Lake/Apache Hudi), 매니지드 서비스(Databricks, Athena, Glue)로 경계가 흐려진다.

```
   Operational (upstream)            Analytical (downstream)
   +------------------+              +---------------------+
   | OLTP / Source DB |   --ETL-->   | Warehouse / Lake    |
   | low latency      |              | high throughput     |
   +------------------+              +---------------------+
                                        |          |
                                  schema-on-write  schema-on-read
                                   (Warehouse)        (Lake)
                                        \          /
                                       +-------------+
                                       |  Lakehouse  |
                                       +-------------+
```

### Syncing Data Between Warehouses and Lakes
- 데이터 통합 계층(AWS Glue, Fivetran, Matillion)이 서로 다른 저장소를 연결한다.
- ETL(Extract-Transform-Load): 추출 → 변환 → 적재 순으로 통합하는 대표 프로세스.

---

## 3. Collecting Data Quality Metrics (데이터 품질 메트릭 수집)

> "측정할 수 없으면 고칠 수 없다." 데이터 품질 메트릭은 데이터가 건강하고 신뢰 가능한지 알려주는 KPI(Key Performance Indicator)이다.

### What Are Data Quality Metrics?
- 데이터 다운타임(부분적·오류·누락·부정확한 기간) 관점에서 품질을 측정할 것을 권장한다.
- 일부 기업은 SLA(Service-Level Agreement)로 데이터 팀에 책임을 부여하나 아직 표준은 아니다.
- 핵심 점검 질문: 최신성, 완전성, 필드 값 범위, null 비율, 스키마 변경 여부.

### How to Pull / Example: Snowflake (4단계)
- Step 1 — Map inventory: information_schema.tables로 테이블·스키마 목록과 메타데이터 매핑.
- Step 2 — Freshness/Volume: ROW_COUNT, BYTES, LAST_ALTERED로 최신성·볼륨 추적, 이상 업데이트 탐지.
- Step 3 — Query history: query_history로 lineage·사용자·비용·성능 분석.
- Step 4 — Health check: 문자열 필드는 completeness/distinctness/UUID rate, 숫자 필드는 zero rate/mean/quantiles 추적.
- 확장성: 최근 데이터(예: 1일)만 추적+샘플링하고 과거치는 저장하여 비용 효율화(Snowflake 크레딧 주의).

### Using Query Logs (웨어하우스 & 레이크)
- 쿼리 로그는 "누가 접근?·상류/하류 출처?·실행 빈도?·영향 행 수?" 질문에 답한다(Snowflake QUERY_HISTORY, BigQuery AuditLogs, Redshift STL_QUERY).
- 로그는 보존 기간이 짧으므로, 필요한 메트릭을 영구 저장소에 사전 적재해야 한다.
- 레이크에서는 스키마 강제가 없어 메트릭 확보가 어렵지만, 객체 삽입 시각·크기·포맷·암호화 여부 등 시스템 메타데이터를 활용할 수 있다.

---

## 4. Designing a Data Catalog (데이터 카탈로그 설계)

> 데이터 카탈로그는 도서관 목록과 유사한 메타데이터 인벤토리로, 데이터 접근성·건강·위치를 평가하는 단일 진실 공급원이다.

- 답하는 질문: 어디서 찾는가? 중요한가? 무엇을 의미하는가? 어떻게 쓰는가?
- 전통적으로 Excel 수기 관리였으나, 수만 개 테이블·비구조화 데이터 시대엔 자동화가 필수.
- Alation, Collibra, Informatica 등은 ML·자동화로 발견성·협업·규정 준수를 강화한다.

### Building a Data Catalog
- 다운스트림 이해관계자와 정렬 → 중요 데이터 식별 → 소유자(owner) 지정 → 카탈로그 채우기.
- SQL 파서(Sqlparse, ANTLR, Apache Calcite)로 SQL을 자료구조로 분해하여 메타데이터 추출.
- 저장: ELK 스택, PostgreSQL, MySQL, MariaDB / 시각화: Amundsen, Apache Atlas, DataHub, CKAN.

### Data Discovery (데이터 디스커버리)
- 데이터 메시(Zhamak Dehghani) 기반으로, 카탈로그의 "이상적 상태"가 아닌 데이터의 "현재 실시간 상태"를 도메인별로 파악한다.
- 품질 우선 카탈로그 특징: 셀프서비스 발견·자동화, 데이터 진화에 따른 확장성(ML 활용), 분산 디스커버리를 위한 데이터 리니지(lineage).

```
  +---------------------------------------------+
  |            Data Catalog (metadata)          |
  |  table | report | last_updated | owner |... |
  +---------------------------------------------+
            ^                         |
     SQL Parser (ANTLR)         Data Discovery
     (extract metadata)     (real-time current state
                              per domain, ML-driven)
```

---

## Summary (핵심 정리)

- 진정으로 발견 가능한 데이터는 단순 "카탈로그화"를 넘어 정확·정제·완전 관측 가능, 즉 신뢰 가능(reliable)해야 한다.
- 데이터와 그 상태, 사용 방식을 라이프사이클 전 단계·전 도메인에서 이해할 때 비로소 신뢰가 시작된다.
- 다음 장에서는 파이프라인 전반의 데이터 수집·정제·변환·테스트 기본기를 다룬다.
