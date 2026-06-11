# 07. Monitoring Data Against Rules

## 챕터 개요 (3줄 요약)

- Monitoring은 rule을 3단계 report 계층(high-level dashboard → Rule Results Report → Failed Data Report)으로 조직해 조직이 action을 취하게 한다.
- 각 report는 seniority별 stakeholder(data owner·steward·producer)를 대상으로 하며 drill-through로 연결된다.
- inactive·duplicate 데이터를 별도 methodology로 필터링하고, report를 성공적으로 launch·governance에 embed해야 한다.

---

## 1. Reporting 계층 (Table 7.1)

> high-level 요약부터 개별 failed row까지 hierarchy 제공. 사용자는 보통 3단계를 모두 traverse(drill-through).

```text
High-level Dashboard   → senior(data owner): 영역 진척 관찰·자원 배분
Rule Results Report    → data steward: rule별 점수·우선순위화 (가장 널리 사용)
Failed Data Report     → data producer: 교정 to-do 목록
(별도) Inactive/Duplication Report → 위 계층을 filter
```

### Data Security
- row-level 상세는 PII·protected characteristic·confidential(급여·생년월일) 노출 위험 → SME·GDPR·IT security 리뷰로 sensitive 필드 식별·보호. VIP 고객/임원 row 제외 가능.
- **권장: 가능한 open하게 두고 진짜 sensitive만 숨김** → healthy competition 유발(사례: Spain·Portugal 경쟁으로 선두 등극).

---

## 2. High-Level Dashboard

> 가장 바쁜 senior 리더용 → 단순하게. process area·data object·business unit·region별 요약 + business KPI 연결.

- dimension: 조직 특성에 맞게(product-centric이면 product 추가, 공장별 등). 항상 시간 trend(보통 18개월).
- filter(Table 7.2): person responsible(저자 비권장-blame culture), subprocess, country, source system, data object 속성, date range, dimension.
- 시각: summary matrix(object×region, 색상·개선 icon), score vs target, dimension별 trend line, region별 비교.
- 분석 예: completeness는 빨리 개선되나 validity는 뒤처지는 흔한 패턴(양 vs 질 균형) → communication으로 재조정.
- **Drill through**: matrix 숫자/line에서 Rule Results Report로(필터 유지).

---

## 3. Rule Results Report (가장 널리 사용)

> high-level trend → 구체적 rule별 이슈 목록. data steward·owner·운영진의 공통 언어.

- 정보: rule name, region·dimension, passed/failed/total, 점수(전월·당월), trend.
- **Color-coding (RAG)**: rule별 low/high threshold 기준. 직관에 반할 수 있음(예: 점수 98이나 high threshold 99면 amber, 점수 90이나 high 88이면 green) → training 필요. 색맹(deuteranopia) 접근성 고려.
- 추가 시각: rule 점수 시간 trend(threshold 대비), 국가별 failed record 수.
- 분석 예: Africa 전체는 개선되나 Nigeria VAT rule은 악화 → table/trend만으론 못 봄, drill로 발견.
- 활용: filter로 outlier 탐색, bookmark로 중요 조합 저장, high threshold 위 negative trend 선제 대응, Failed Data Report로 drill해 상세 rule 정의 확인.

---

## 4. Failed Data Report

> 최종 상세 — 교정 가능한 actionable record 목록. 조직 간 거의 차이 없음.

- 목적: record 식별, 이슈 표시, 교정 지원(예: 상세 rule logic, supplier ID/name/country, 관련 ID(VAT·DUNS), **연락처(email/phone)** — system of record 재방문 불필요).
- **Re-use**: 같은 보조 컬럼이면 여러 rule이 한 report 공유(예: VAT·DUNS). conditional formatting은 reactive(검사 대상 컬럼만 강조).
- **Multiple reports**: supplier record의 다른 부분(예: bank details)은 다른 컬럼 → 단일 report면 컬럼 과다·느림·security 위험. rule 수보다 적은 수의 report.
- **Export**: 전체 suite는 interactive 권장이나 Failed Data Report는 예외(to-do 추적, Excel/SAP LSMW 업로드). export 전 region·BU·data type filter로 행 축소 권장.

---

## 5. Managing Inactive & Duplicate Data

> 비활성·중복 record 교정에 시간 낭비 방지. remediation 작업량 감소.

### Inactive 데이터
- methodology는 data steward가 수립, 데이터별 상이, iterative.
- Supplier(Table 7.4): last order ≤13개월, last payment ≤7개월, open transaction, record 생성 ≤3개월 → Active.
- Product(Table 7.5): last order/shipment ≤25개월, 제조 ≤6개월, inventory, 생성 ≤3개월 → Active.
- ETL로 transactional 데이터를 master에 join해 Active/Inactive flag → report에서 inactive 무시(사용자가 파라미터 조정 가능).

### Duplicate 데이터
- 생성 시 중복 탐지 process 부재·결함, 또는 migration 부실로 발생.
- Supplier 매칭 필드+가중치(Table 7.6): DUNS 30, VAT/tax 25, 등록번호 25, 우편번호 15, 주소 10, 이름 10 → match confidence %.
- name/address만으론 불신뢰(공유 본사). **fuzzy matching**(완전 일치 아니어도). 높은 confidence는 자동 merge, 낮으면 manual review.
- **중복을 먼저 처리** 권장 → remediation 작업량↓, 제3자 인식·고객 서비스 개선(단일 account view).

---

## 6. Presenting Findings & Governance

> rule이 정확·잘 test되어야 launch 성공. false failure는 engagement를 무너뜨림.

- 초기엔 engaged steward/producer에게 먼저 release(1-2주 모니터)→champion화. 접근을 frictionless하게(사전 account·license).
- **자동 배포**·**alert**(예: P2P 점수 <85% 시 process owner에 이메일).
- **Governance embed**: exec committee(분기), regional/functional leadership(분기), data steering committee(월간) 표준 agenda. 직원 objective에도 target 반영.

---

## Summary (핵심 정리)

- 누구나 전체 data quality 그림과 trend를 보고, 원하면 record level까지 drill할 수 있는 monitoring report suite를 제시했다.
- report는 입력된 rule만큼만 좋으며, Failed Data Report가 remediation할 actionable to-do를 제공한다.
- 다음 챕터(8장)는 이 목록으로 데이터를 정리(remediation)해 Ch3에서 식별한 benefit을 실현하는 것을 다룬다.
