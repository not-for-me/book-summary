# 02. The Principles of Data Quality

## 챕터 개요 (3줄 요약)

- Data quality는 data management(특히 data governance) 안에 위치하며, ownership·definition·catalog·data model·CRUD process·policy·tools와 symbiotic하게 작동한다.
- DAMA UK의 6개 data quality dimension(completeness·uniqueness·timeliness·validity·accuracy·consistency)과 핵심 용어를 정의한다.
- 데이터 quality 개선은 hub-and-spoke stakeholder 모델과 6단계 data quality improvement cycle(business case → discovery → rule development → monitoring → remediation → BAU)로 진행된다.

---

## 1. Data Quality in Data Governance

> Data governance = 데이터가 의도된 목적에 fit-for-purpose하도록 보장하는 people·policy·process·tool의 집합. data management의 일부이자 foundation.

- Data management 범위: data quality, data privacy, MDM, data warehousing/BI (DAMA DMBoK 참조).
- **Data ownership**: 특정 entity(예: customer)에 accountability를 갖는 개인. 개선의 "driving force".
- **Data definitions**: data catalog에 상세·specific하게 존재해야 함. 예제(엔진 제조사): CRM에 email 필드 4개지만 정의는 5개(direct sales의 Email 1 용도가 corporate와 달라 별도 정의). marketing팀이 Email 2만 보고 completeness 3.8%로 오인 → Email 1 포함 시 91%.
- **Knowledge of how data is used**: owner는 consumer와 대화, 사용 정책을 catalog에 반영. 예제: marketing consent flag·국가별 법규 확인 후 연락 가능 비율 58%로 하락(privacy 위반 회피).

### Data Catalog 구성요소

```text
Field status      mandatory / business-mandatory / optional
Data domain       소유 영역 (예: customer = sales)
Data standard     필드 캡처 방식 (예: 법적 등록명)
Field length/type 길이·데이터 타입 (보통 system-agnostic, metadata tool이 스캔)
```

- catalog는 Wikipedia처럼 누구나 변경 제안 → steward가 승인·반려해야 번성.

### 기타 governance 요소

- **Data model**: system-agnostic conceptual model. entity(customer·order·product 등)와 relationship(1:N). 예: Email 4는 customer가 아닌 delivery address entity에 속함.
- **CRUD process**: Create/Read/Update/Delete. 역할·training·validation·approval·다중 system·duplication check 고려. 속도 vs governance 균형(Ch1 Fig 1.3).
- **Data policies**: duplication, active/inactive, retention, data quality 정책. → data quality rule의 input.

---

## 2. Data Governance Tools & MDM

> Tool은 people·process·policy와 함께일 때만 가치. ownership 없이 DQ tool만 도입하면 우선순위·자원·budget 확보 실패.

- **MDM tools**: master data 생성·변경 자동화·validation·통합. golden record(single source of truth).
  - **Central MDM**: 전체 생성/갱신을 MDM이 semi-automated 관리, 다중 수신 시스템에 replicate. 최근엔 SAP Ariba·Shopify 등이 1차 수집 후 MDM에 전달하는 역방향이 흔함.
  - **Consolidation**: 여러 system of record의 데이터를 matching해 공통 ID 부여, golden record 구성.
- **Metadata management tools (data catalog tools)**: 데이터에 관한 모든 것 저장(데이터 자체 제외). Microsoft Purview, Informatica, Collibra, Alation.
  - business glossary + technical catalog 두 capability를 연결하는 것이 핵심 (예: SAP MARA/MATKL ↔ business term "product type").
  - 연결되면 report column에서 hotkey로 definition·owner·lineage·**현재 data quality 수준**을 즉시 표시.

### DQ ↔ Governance touchpoint (Table 2.6 요약)

- ownership → 우선순위·accountability·자원 제공 / DQ → 개선·악화의 정량 view
- definitions → rule 생성 가속 / DQ → definition 정제
- data model → 의존성 표시 / DQ → model 개선 정보
- catalog → 정의·owner·lineage로 시간 절약 / DQ → catalog enrich

---

## 3. DAMA 6 Data Quality Dimensions

> Data quality = 데이터가 의도된 목적에 사용될 수 있는 정도. dimension은 rule을 theme으로 묶고 광범위한 rule 검토를 prompt함.

```text
Completeness  '100% complete' 대비 저장된 데이터 비율 (가장 이해 쉬움, but 과집중 금물)
Uniqueness    동일 thing이 한 번만 기록 (duplication 정책으로 식별 정의)
Timeliness    필요 시점 기준 reality 반영 정도 (예: credit check 3개월 이내)
Validity      정의된 syntax(format/type/range) 준수 (실재 여부는 미확인)
Accuracy      real-world 정확 기술 — authoritative source 비교 필요 (예: D&B)
Consistency   여러 표현 간 차이 부재 (예: 시스템 간 employee 데이터 sync)
```

- **Validity vs Accuracy**: `rob@hawker.com`은 valid(형식)지만 accurate하지 않음(실재 도메인 아님). accuracy rule은 3rd-party 자원 필요로 비싸고 어려움.
- 대부분 조직은 completeness·validity에 rule 편중(쉽기 때문) → 모든 dimension 균형이 더 가치. 한 dimension 누락 시 중요 rule 놓침.

### 핵심 용어 (Table 2.7)

- **Data quality rule**: 각 row에 적용해 correct/incorrect 판정하는 logic.
- **Failed/passed records**, **data quality issues**(영향 분석·resolution), **data profiling**(값·길이·completeness·분포 통계), **monitoring**, **remediation**, **data domains**.

---

## 4. Stakeholders (Hub-and-Spoke 모델)

> Hub(중앙, 정부처럼 framework 작성)가 vision·strategy 설정, Spoke(business unit)가 해석·실행·집행.

### Hub roles (full-time)
- **CDO**: data governance·analytics 전체 accountability (없으면 CTO/CFO가 대행).
- **Data governance lead**: governance 전략, owner 배치·engagement.
- **Data quality lead**: DQ 전 단계 주도.

### Spoke roles
- **Data owner** (part-time, senior): 특정 domain의 governance·quality 전적 accountability.
- **Data steward** (full/part-time): owner가 임명, day-to-day 실무(정의 수집·rule 정의·process 문서화).
- **Data champion** (part-time): cross-domain 일관성 견인(예: 중복 정책 통일).
- **Data producer**: 실제 데이터 생성자. **Data consumer**: 데이터 사용자. **Process lead**: process 영향 통찰.

---

## 5. Data Quality Improvement Cycle (책 전체 구조)

> 일부 데이터(supplier·employee 등)를 선택해 cycle 전체를 완료하는 반복 구조.

```text
Business case   → 지원·budget 확보, 비용/효익 추정 (Ch3)
Data discovery  → strategy↔process↔data 연결, profiling으로 상세 scope (Ch5)
Rule development→ profiling+workshop으로 rule 도출, tool에 구현·test (Ch6)
Monitoring      → stakeholder별 report로 결과 전달·trend (Ch7)
Remediation     → 우선순위화 후 데이터 교정, forum governance (Ch8)
Embedding BAU   → 일상 운영 지속, 원인 제거(training), rule 최신화 (Ch9)
```

- 필수 단계: rule development, monitoring, remediation, embedding BAU. business case·discovery는 일부 cycle에서 생략 가능.

---

## Summary (핵심 정리)

- 모든 독자가 공통으로 이해해야 할 data management·data governance 개념과 DQ의 위치를 정립했다.
- DQ 성공은 모든 level stakeholder의 관심·지원에 의존하며, hub-and-spoke 역할들이 이를 뒷받침한다.
- 다음 챕터(3장)는 가장 어려운 단계인 business case for data quality를 다룬다 (대부분 이니셔티브가 여기서 실패).
- 참조: DAMA UK 백서 《The six primary dimensions for data quality assessment》.
