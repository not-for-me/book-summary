# 03. Collecting, Cleaning, Transforming, and Testing Data

## 챕터 개요 (3줄 요약)

- 데이터를 프로덕션 사용에 대비시키기 위한 4단계(수집·정제·변환·테스트)를 통해 파이프라인 전반의 데이터 품질을 관리하는 법을 다룬다.
- 배치 vs 스트림 처리의 차이, AWS Kinesis/Apache Kafka 활용, 정규화·스키마 검사·타입 강제 등 운영/분석 변환 단계의 품질 확보법을 설명한다.
- dbt·Great Expectations·Deequ 테스트 도구와 Apache Airflow의 SLA·서킷 브레이커·SQL 체크 오퍼레이터로 사전 예방적 품질 관리를 구현한다.

---

## 1. Collecting Data (데이터 수집)

> 엔트리포인트(entrypoint)는 외부 세계의 데이터가 파이프라인에 처음 진입하는 가장 상류 지점으로, 가장 raw하고 노이즈가 많다.

- 데이터는 보통 이질적(heterogeneous, 구조화+비구조화)이며 소스 선택은 비즈니스 목표/상류 도구에 의존한다.
- 소스는 크게 세 범주: 애플리케이션 로그, API 응답, 센서 데이터.

### Application Log Data
- 소프트웨어 동작으로 생성되며, 무엇을 기록할지는 개발자가 결정(완전한 이력 아님).
- 고려사항: 구조(ASCII/binary), 타임스탬프(ISO 표준), 로그 레벨(INFO/WARN/ERROR), 목적(진단 diagnostics / 감사 auditing).

### API Responses
- API(Application Programming Interface)는 두 프로그램 간 중개자로, 정해진 형식의 요청/응답을 주고받는다.
- 고려사항: 구조(JSON 등 키-값/리스트), 응답 코드(HTTP 200/404/500), 목적(코드 vs 본문 중 무엇이 중요한지).

### Sensor Data
- IoT(Internet of Things)·연구 장비 등에서 수집되며 매우 노이즈가 많다.
- 고려사항: 노이즈(다운스트림 정제 필요), 실패 모드(고장 시 알림 없이 이상값 전송), 목적(ML은 throughput, 추론은 latency 중요).

---

## 2. Cleaning Data (데이터 정제)

> 데이터 정제는 사용 가능한 데이터셋에서 부정확/비대표적 데이터를 제거하는 작업으로, 고품질 데이터의 최대 난관이다.

- Outlier removal: 표준 점수(standard scoring), isolation forest 등으로 이상값 제거(시간 복잡도 주의).
- Assessing features: 다운스트림에 불필요한 필드는 제거(과도한 피처는 문서화·도메인 지식 부담 증가).
- Normalization: L1(Manhattan), L2(Unit) Norm, demeaning, unit variance 등 용도별 선택.
- Data reconstruction: 보간(interpolation)·외삽(extrapolation)으로 결측값 복원.
- Time zone conversion: UTC(Coordinated Universal Time) 표준으로 변환(시간대 혼동은 버그의 흔한 원인, 예: Y2K).
- Type coercions: 다운스트림 요구 타입으로 캐스팅(예: float 4.99→int 4는 절삭, 주의).

---

## 3. Batch Versus Stream Processing (배치 vs 스트림 처리)

> 배치는 일정 기간 데이터를 모아 처리하고, 스트림은 거의 즉시 처리한다. 핵심 차이는 배치당 데이터량과 처리 속도이다.

- 2010년대 중반까지 배치가 표준(저렴, 충분). 실시간 수요 증가로 Kafka·Kinesis가 스트리밍을 대중화.
- 예시: 신용카드 결제 정산은 배치, 사기 탐지·차량공유 매칭은 스트림.
- 배치 프레임워크: Apache Hadoop(파일 분할·노드 분산), 매니지드: BigQuery/Snowflake/Azure/Redshift.
- 스트림 기술: Apache Spark(마이크로배치)·Kafka(near real-time)·Flink/Storm/Samza, 매니지드: Databricks/Cloudera/Azure.

### Data Quality for Stream Processing
- 배치는 품질이 높은 편, 실시간 스트림은 손실(lossiness)로 오류 여지가 커진다.
- 전통적 테스트는 확장이 어렵고 가능한 품질 이슈의 약 20%("known unknowns")만 커버.
- 실시간 데이터는 단위·기능·통합 테스트만으로 확장 불가 → 접근 방식 재고 필요.

### AWS Kinesis vs Apache Kafka
- Kinesis: 서버리스, 온디맨드 확장, 비용 효율, 다양한 SDK(Java/Go/.NET), AWS 통합 용이 / 보존 7일 고정.
- Kafka: 오픈소스, 높은 커스터마이징(보존기간 조정), 높은 throughput(~30,000 records/sec), 낮은 지연(~2ms).
- 소규모/빠른 도입은 Kinesis(SaaS), 대규모/특수요건은 Kafka(오픈소스)가 적합.

```
   Sources (logs/API/sensors)
            |
   +------------------+        +-------------------+
   |  Stream (Kinesis/|  --->  |  Lake (raw/flat)  |
   |  Kafka, low lat) |        |  schema-on-read   |
   +------------------+        +---------+---------+
                                         | normalize / ETL (Glue, Lambda)
                                         v
                               +-------------------+
                               | Warehouse (struct)|
                               +-------------------+
```

---

## 4. Normalizing Data (데이터 정규화)

> 정규화는 여러 소스 포맷을 목적지 포맷으로 옮기는 첫 변환 단계로, 노이즈·이질성이 최대인 엔트리포인트 데이터를 다룬다.

- 정규화 시점 데이터 특징: 지연 최적화(불완전 배치), 비계층적 flat 포맷(S3 dump), raw 파일 포맷, 선택적 필드, 이질성.
- 레이크가 진입 데이터의 선호 저장소(제약이 적음) → 스트리밍이 레이크에 적재 후 변환으로 웨어하우스로 승격.

### Schema Checking and Type Coercion
- 스키마 검사: 필수 필드 존재·형식 검증. 스키마 변경은 데이터 깨짐의 주요 원인 → 예상 스키마 기록·변경 추적.
- 타입 강제: 암묵적 캐스팅이 위험할 수 있음(유효숫자 손실, 정수 절삭).

### Syntactic vs Semantic Ambiguity
- 구문적 모호성(syntactic): 같은 지표가 다른 필드명/타입으로 표현됨(마찰 유발).
- 의미적 모호성(semantic): 같은 필드의 목적에 대한 합의 부재(핵심 지표 왜곡 위험) → 사전 문서화가 핵심.

---

## 5. Running Analytical Data Transformations (분석 데이터 변환)

> 분석 변환은 분석 데이터에 대한 변환으로, ETL(추출-변환-적재) 과정에서 품질을 보장한다.

- ETL: Extract(상류 소스에서 스테이징으로) → Transform(결합·처리) → Load(목적지 테이블 적재).
- ETL은 프로덕션 전 검증 가능, ELT(추출-적재-변환)는 빠르지만 테스트 없으면 품질 저하.
- 변환 목적: 필드 리네이밍, 필터·집계·중복제거, 타입·단위 변환, 암호화, 거버넌스/품질 점검.

---

## 6. Alerting and Testing (알림과 테스트)

> 테스트는 데이터가 프로덕션에 들어가기 전 품질 이슈를 발견하는 과정으로, 데이터에 대한 가정을 검증한다.

- 흔한 테스트: null 값, 볼륨, 분포(distribution), 유일성(uniqueness), 알려진 불변식(known invariants).
- 절차: 변환 데이터를 임시 스테이징 테이블에 적재 → 임계값 충족 여부 테스트 → 실패 시 알림·파이프라인 중단.

### dbt Unit Testing
- dbt run은 SQL 변환 실행, dbt test는 단위 테스트 실행. 테스트는 "실패 행"을 반환하는 SQL 패러다임.
- 두 종류: Singular tests(특정 모델용 독립 SQL), Generic tests(파라미터화·재사용). 기본 제공: unique, not_null, accepted_values, relationships.
- 한계: 수동 유지보수 기술부채, 테스트 피로(무의미한 테스트), 제한된 가시성(엔드투엔드 아님).

### Great Expectations Unit Testing
- Python으로 작성, ETL/ELT에 폭넓게 적용. "expect_column_values_to_be_between" 등 라이브러리 제공.
- 결과를 "Data Doc"(사람이 읽는 페이지)로 렌더링. 장점: 사용 용이(Python/Jupyter), Slack 연동.
- 한계: Python 한정, 변환/오케스트레이션 도구와 분리됨.

### Deequ Unit Testing
- AWS가 Apache Spark 위에 구축한 오픈소스(Scala), PyDeequ도 제공. VerificationSuite로 검사 추가.
- 장점: AWS 통합, 높은 확장성, 상태 기반(stateful) 증분 계산, 내장 이상 탐지.
- 한계: Scala 학습곡선, 통합 테스트엔 제한적, 직관적 UI 부재.

---

## 7. Managing Data Quality with Apache Airflow (Airflow로 품질 관리)

> Airflow는 오케스트레이션 계층에서 워크플로우를 작성·스케줄·모니터링하며, DAG(Directed Acyclic Graph)의 체크포인트로 품질을 관리한다.

- 흔한 다운타임: 악화되는 쿼리(스케일 한계), 버그 있는 Python 코드.

### Scheduler SLAs
- 태스크에 datetime.timedelta로 SLA(Service-Level Agreement) 설정, 초과 시 "SLA missed" 표시 또는 sla_miss_callback 트리거.
- 콜백 5개 파라미터: dag, task_list, blocking_task_list, slas, blocking_tis.

### Circuit Breakers (서킷 브레이커)
- 데이터가 품질 임계값을 못 넘으면 파이프라인을 중단(CI/CD의 서킷 브레이킹 차용).
- 두 상태: Circuit closed(데이터 흐름) / Circuit open(흐름 중단).
- 3대 요소(Sandeep Uttamchandi): 데이터 리니지, 파이프라인 전반 프로파일링, 프로파일링 기반 자동 트리거.
- Airflow 적용: catchup=False, LatestOnlyOperator, 커스텀 Python 코드 삽입.

```
   [Upstream Job] --> ( Quality Check )
                          |        |
                       PASS      FAIL
                          |        |
                   circuit closed  circuit open
                   (flow down)   (stop pipeline)
```

### SQL Check Operators
- DAG/파이프라인 전반에서 값·구간·임계값을 검증(Great Expectations·dbt와 유사). 단일 행 반환으로 False 여부 확인.
- 주의: 파이프라인 중단은 심각한 인시던트에만 사용(무관한 고품질 작업까지 막을 수 있음).

---

## Summary (핵심 정리)

- 데이터 다운타임은 수집부터 BI 계층까지 각 단계에 품질 점검을 통합하여 사전 예방적으로 다뤄야 한다.
- 기술만으로 품질을 완전히 해결할 순 없지만, 신뢰성을 염두에 둔 수집·정제·처리·오케스트레이션이 큰 도움이 된다.
- 완벽한 신뢰성은 불가능하므로(unknown unknowns 존재), 다음 장에서 이상 탐지(anomaly detection)로 보완한다.
