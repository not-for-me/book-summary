# 06. Data Quality Rules

## 챕터 개요 (3줄 요약)

- Data quality rule은 각 row에 적용해 pass/fail(Boolean)을 판정하는 logic이며, false positive를 막는 rule scope이 핵심이다.
- rule의 key feature: weighting, dimension, priority, threshold(low/high RAG), cost per failure를 설계 시 고려해야 한다.
- 구현은 design(business→technical) → build(connect·ETL·rule·report) → test의 IT implementation 과정이며, 반복(iterate)과 결함 검출이 중요하다.

---

## 1. Introduction to Data Quality Rules

> Rule은 항상 Boolean(pass/fail) 출력. 개별이 아닌 amalgamation으로 전체 품질 그림과 failed record 목록을 제공.

### Rule Scope (가장 중요)
- generic rule은 임팩트 약함 → scope으로 specific하게. 예: "Services category supplier만 payment term ≥ 30일" (utility 등 제외해 false positive 방지).
- VAT rule: UK suppliers로 한정해야(타국 형식 상이로 전부 fail 처리됨). turnover 임계(예: UK £80,000) 이상만 포함.
- workshop에서 stakeholder는 scope 없이 rule 제시 → challenge로 scope 보완. scope 차이가 평가 record 수를 결정(예: 3,000명 중 contractor 400명만 end date rule).

---

## 2. Key Features (Fig 6.1)

### Rule Weightings
- 중요 rule에 가중치 부여해 전체 score에 영향(예: 가중 1.5). 저자는 **비선호**(투명성↓, 설명 부담, 도구 신뢰 의심 유발).
- 유용한 경우: ① 데이터셋 사용 성패를 가르는 소수 rule(예: bonus 계산 5개 critical rule > 편지용 rule), ② 규제 산업 compliance rule. 단, 별도 report 분리가 대안.

### Rule Dimensions
- 6개 dimension(completeness·uniqueness·timeliness·accuracy·validity·consistency)으로 수백 rule을 그룹화 → leadership 대화 가능(예: "product completeness Q1 70%→Q2 64%"). 단, 전략적 theme(예: bonus)으로 묶는 게 나을 때도.

### Rule Priorities (Table 6.5)
```text
Critical  1개월 내만 허용 / 직접 영향 ≥ \$100k / 즉시 규제 위반 / 주요 파트너 관계 붕괴 / 평판 회복불가
High      2개월 / ≥ \$50k / 규제 위반 위험↑ / 관계·평판 손상
Moderate  6개월 / ≥ \$10k / 관계 부정적
Low       그 외 (판단으로 상향 가능)
```

### Rule Thresholds (가장 critical)
- bad data 정의를 넘어서는 기준선 설정. 보통 **2개 threshold(low/high) → RAG**:
```text
score < low          critically poor (red)
low ≤ score < high   moderately below (amber)
score ≥ high         expected standard (green) — 추가 노력 불필요
```
- rule마다 다르게 설정: zero tolerance(employee bank details = 100%, payroll cutoff 전 입사자로 scope 한정), lower tolerance(consumer email 60%/80% — 단 저자는 account 없는 고객 제외로 scope 조정 선호).
- 시작 score가 매우 낮으면(30%) 동기부여 위해 50%/70%로 설정 후 재조정 가능(gray area). aggregated threshold(dimension·business unit·region)도 정의 — 예: critical rule이 red면 전체 red.

### Cost per Failure
- 실패당 비용 부여(예: \$100 × 50,000 = \$5M). 강력하나 **신중히** — 추측성이면 scrutiny에 즉시 discredit(예: DUNS 누락 → 할인 가정은 "내년에도 거래?" 질문에 무너짐).
- 유효한 경우: 주관성 없는 경우(신규 입사자 IT account 데이터 = \$400/day, 누락 license = 규제 벌금 \$1,000). criticality·threshold 산정에 기여 가능.

---

## 3. Implementing Rules (design → build → test)

> 다른 IT 구현과 유사하나, **iterate 준비**가 고유 — design 확신해도 build/test에서 예상 못한 subtlety 발견.

### Designing Rules
- **Business language description**: profile 결과를 stakeholder에 제시 → "what good looks like" 도출. scope + constraint(상하한/패턴) + 다른 필드 의존성 + **business reason**(추가 아이디어 유도, 예: product weight rule이 중간 상태 category까지 확장).
- **Additional info**: weighting, threshold, dimension, priority, cost per failure, technical field명.
- **Technical design** (solution architect): business → system 번역.
```text
예) product weight (category X,Y) 0.10~0.20 kg
    Table: MARA, Field: NTGEW (net weight)
    Filter: MARA-MATKL = A101, A102
    pass if NTGEW >= 0.1 and <= 0.2 else fail
```
- 복잡도 평가 후 최종 scoping(budget·시간에 맞춤).

### Building Rules (Table 6.10)
```text
Connect to sources    background user account 생성 (leader가 access 지원)
ETL jobs              다중 source 통합·schedule (가장 기술적, 지원 최소)
Develop rules         profile에서 직접 생성 또는 from scratch (모호성 질문에 즉답 필요)
Integrate reporting   failed data report 매핑 (Ch7)
Move to test          change board 승인
```
- **Early build visibility**(2주마다 확인, 계약에 명시), **representative data**(dev 시스템은 live의 ~5% → live copy 제공, 민감정보 scramble).

### Testing Rules (2-phase)
- ① 개별 rule test(integration/product test, 전담 인력), ② report 전체 결과 test(user acceptance, end user=data steward, 사전 training).
- **Data sheets**: business analyst가 사전에 SAP에서 추출·필터한 Excel로 aggregate 결과(총 passed/failed) 검증 (test 기간 중 데이터 불변 전제).
- **Multiple levels of review**: tester가 결함 놓치기 쉬움(business context 부족·단조로움). 사례: 200 rule 중 tester 20개 + SME 리뷰로 40개 추가 검출. **경험 많은 SME의 전체 리뷰 권장** — live tool의 결함은 신뢰를 무너뜨림(ERP와 달리 무시 가능하므로).

---

## Summary (핵심 정리)

- false positive를 막는 tight scope으로 유용한 rule을 정의하고, weighting·dimension·priority·threshold·cost per failure 정보를 문서화한다.
- design·develop·test 과정에서 좋은 leadership과 SME 리뷰가 결함을 줄여 도구 신뢰를 지킨다.
- 다음 챕터(7장)는 이 rule들이 생성한 결과를 stakeholder에게 제시하는 monitoring/reporting을 다룬다.
