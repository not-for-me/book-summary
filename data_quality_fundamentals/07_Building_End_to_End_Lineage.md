# 07. Building End-to-End Lineage

## 챕터 개요 (3줄 요약)

- 데이터 리니지(lineage)를 데이터의 라이프사이클 여정 지도로 정의하고, 근본원인·영향 분석의 핵심 도구로서 필드 수준(field-level) 리니지를 직접 구축하는 법을 다룬다.
- 리니지의 기본 요구사항·데이터 모델 설계·SQL 쿼리 파싱(ANTLR)·UI 구축 단계와 모범 사례를 설명한다.
- Fox의 "controlled freedom" 셀프서비스 데이터 플랫폼 사례로 리니지·옵저버빌리티 기반 데이터 신뢰 구축을 보여준다.

---

## 1. Building End-to-End Field-Level Lineage (필드 수준 엔드투엔드 리니지 구축)

> 데이터 리니지는 데이터가 A지점에서 B지점으로 어떻게 이동했는지의 기록으로, 상류 소스와 하류 의존성 간 관계를 추적한다.

- SRE(Site Reliability Engineering)의 git blame처럼, 리니지는 데이터가 어디서 깨졌는지 맥락 속에서 파악하게 해준다.
- 테이블 수준(table-level) 리니지는 거시적이나, 파이프라인이 왜/어떻게 깨졌는지 알기엔 입자도가 부족 → 필드 수준 필요.

### Basic Lineage Requirements (기본 요구사항)
- Fast time to value: 필드 수준으로 추상화하여 빠른 영향 파악.
- Secure architecture: 사용자 데이터·PII(Personally Identifiable Information) 직접 접근 회피, 메타데이터·로그·쿼리만 활용(데이터는 고객 환경에 유지).
- Automation: 수동이 아닌 자동 업데이트 방식 권장.
- Integration: 웨어하우스/레이크(Snowflake, Redshift, Databricks), 변환(dbt, Airflow, Prefect), BI(Looker, Tableau, Mode) 통합.
- Extraction of column-level info: 쿼리 로그 파싱만으로는 부족, 컬럼 수준까지 추출.

### 분석 활용 사례
- 매출 리포트의 의심스러운 수치 추적(원인: 실행 실패한 dbt 모델).
- 데이터 부채(data debt) 축소: 사용 안 하는 컬럼 deprecation 안전 확인.
- PII 관리: PII 컬럼이 연결된 하류 대시보드 신속 파악.

---

## 2. Data Lineage Design (리니지 설계)

> 어떤 컬럼이 어떤 소스 테이블에 속하는지 파악하는 것이 핵심이며, 다중 소스·중첩 서브쿼리로 인해 재귀적 해석이 필요하다.

- 리니지 구조 3요소: destination table(하류 리포트 저장), destination fields(목적지 테이블 내), source tables(웨어하우스 저장).
- 유연한 논리 데이터 모델 권장: table_mcon ID + 해시된 필드 수준 리니지 객체를 문서 ID로 사용해 다양한 쿼리 조합 포착.
- Selected fields(결과 테이블을 정의하는 필드) vs Non-selected fields(가져올 행에 영향을 주나 결과값엔 기여 안 함 → UI에서 가려 RCA 단순화).
- 초기 프로토타입은 가능한 SQL 절 조합의 약 70%만 커버 → 각 절을 개별 테스트하며 확장.

```
   Source Tables (warehouse)        Destination Table (report)
   +----------------+               +----------------------+
   | tbl_A: col1    |---SELECT----> | dest_field_1         |
   | tbl_B: col2    |---SELECT----> | dest_field_2         |
   | tbl_C: col3    |---WHERE-----> | (filters rows only)  |
   +----------------+               +----------------------+
        (recursively resolve aliases / subqueries)
```

### Parsing the Data
- ANTLR(ANother Tool for Language Recognition): 구조화 텍스트를 읽고 파싱하는 오픈소스 쿼리파서 생성기.
- 복잡한 쿼리(긴 WITH 절, 중첩 서브쿼리, 따옴표 유무에 따른 컬럼/문자열 구분)에서 성능 이슈 발생 → 파서 문법을 웨어하우스별로 수정.

---

## 3. Building the User Interface (UI 구축)

> 효과적인 리니지는 정보를 그냥 보여주는 게 아니라 "적시에 올바른 정보"를 제공한다.

- 데이터 팀은 보통 가장 하류(BI 객체: Looker/Tableau)나 가장 상류(소스 테이블/필드, 근본 원인) 계층에 가장 관심.
- 재사용 컴포넌트는 JavaScript/TypeScript로 작성, 대규모 렌더링엔 경량 시각화 프레임워크(Apache Preset, React Virtuoso) 활용.
- 두 가지 필드 관계 표시:
  - SELECT clause lineage: 필드-대-필드(상류 필드 변경이 하류 필드를 직접 변경).
  - Non-SELECT lineage: 필드-대-테이블(WHERE 등 필터링/정렬 로직으로 하류 필드 형성).

### 모범 사례
- 팀원의 조언 경청(쿼리파서 난이도 과소평가 사례).
- 프로토타이핑 투자(열성 고객에게 조기 프로토타입 공유로 피드백 가속).
- Ship and iterate(완벽 전에 출시 → 피드백 → 개선 반복).

---

## 4. Case Study: Architecting for Data Reliability at Fox

> Fox Networks의 Alex Tverdohleb가 "controlled freedom" 모델로 셀프서비스 분석을 가능케 한 사례.

### Controlled Freedom (통제된 자유)
- 데이터 팀을 단일 진실 공급원으로 두면 오히려 가장 큰 사일로가 됨 → 게이트키퍼/병목 대신 파라미터만 설정.
- 중앙 팀은 수집·보안·표준 리포트 포맷만 통제, 그 외 디스커버리·ad-hoc 분석은 자유롭게.

### Decentralized Data Team
- 5개 팀(태깅/수집, 엔지니어링, 분석, 데이터 사이언스, 아키텍처)이 협업.
- 분석가는 비즈니스 가까이서 STM(Source to Target Mapping) 작성 → 엔지니어가 명확한 플레이북으로 파이프라인 구축.

### Problem-Solving Tech & Data Trust
- 최신 기술을 좇기보다 비즈니스 이해 우선, 기존 스택 최적화. 레이크하우스로 레이크의 통제력 + 웨어하우스의 정결함 결합.
- "3단계 케이크": raw(원본) → optimized(정렬·최적화) → published(데이터 모델·소비용).
- 일 다회·200+ 소스에서 수신, 주당 ~1만 스키마·수백억 레코드 처리 → 옵저버빌리티가 "사치가 아닌 필수".
- 지속 모니터링·자동 리니지로 프로덕션 전 이슈 포착, 영향 범위 역추적, 비행 중(in-flight) 차단. 투명성이 신뢰의 핵심.

---

## Summary (핵심 정리)

- 리니지는 데이터 파이프라인의 "지도"로, 어느 단계가 다운타임에 영향받든 출발점과 경로를 알려준다.
- 자동화되고 엔드투엔드 커버리지를 제공할 때(달성은 쉽지 않음) 탐지·알림과 결합하여 진정한 데이터 신뢰성의 기반이 된다.
- 리니지가 확장 가능하고 이해관계자가 쉽게 이해할 수 있어야 "controlled freedom"으로 데이터 품질을 넓게 달성할 수 있다.
