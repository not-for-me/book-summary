# 06. Operational Efficiency Cost Model

## 챕터 개요 (3줄 요약)

- 부정확한 데이터의 비용을 직원 compensation(시간) 기준으로 정량화하는 cost model을 제시한다.
- Pre-use validation(DQS 기반, primary control)과 post-use reconciliation(secondary control)의 비용 차이를 manufacturing assembly line 비유로 비교한다.
- Reconciliation을 primary control로 쓰면 이미 손해가 발생하므로, upstream pre-use validation으로 비용을 줄이는 것이 핵심이다.

---

## 1. Model Details & Cost Assumptions

> 부정확한 데이터의 비용 = 직원이 데이터 오류를 추적·수정하는 데 낭비하는 시간(=compensation cost).

- Assembly line: Data Management(저비용 ingest·validation) → Trading → Compliance → Client Reporting(고비용·고영향).
- 이 모델은 잘못된 trade·compliance·regulatory misrepresentation의 재무/평판 손실은 제외하고, 오직 employee time 비용만 다룸 (그 손실은 보통 incalculable).

### 핵심 계산식

```text
business function cost = avg employee cost × number of employees
cost per data element   = business function cost / number of data elements
incorrect data cost     = number of incorrect elements × cost per data element
```

### 예시 입력값 (annualized)

```text
                  DataMgmt  Trading  Compliance  ClientReporting
employees           25       15        10           10
avg cost          25,000   30,000    40,000       75,000
function cost     625,000  450,000   400,000      750,000
data elements      100       80        60           20
cost/element      6,250    5,625     6,667        37,500
```

---

## 2. Pre-Use Validation vs Reconciliation

> Pre-use validation은 데이터 사용 전 upstream에서 downstream DQS 기준으로 품질을 검증(primary). reconciliation은 사용 후 검사(secondary).

- 모든 function에 동일하게 20개 incorrect element 가정. pre-use validation %: Trading 50%, Compliance 70%, Client Reporting 90% (Data Management 0% — 자신이 ingest 주체).
- 비용 절감 예시(20 incorrect elements, 연간):

```text
Function          post-use cost   pre-use 후 잔여   절감액
Trading (50%)     112,500         56,250           56,250
Compliance (70%)  133,340         40,002           93,338
ClientReport(90%) 750,000         75,000          675,000
```

- **누적 pre-use 절감 = $224,588**. 전체 operational efficiency $2,225,000 중, post-use recon은 $995,840 손실, pre-use 적용 시에도 $771,252 손실은 남지만 $224,588을 절감.
- Reconciliation 비유: "cow가 뭘 먹었나" — 거름 분석(reconciliation, 사후) vs 승인된 사료만 급여(pre-use, 입력 통제). 입력을 통제하는 쪽(north end)이 낫고, reconciliation은 secondary 확인용으로만.

> TIP: auditor·regulator는 복수 control 사용을 선호. pre-use=primary + reconciliation=secondary 조합이 audit/regulatory 기준 충족에 유리.

---

## Summary (핵심 정리)

- Pre-use validation은 incorrect data가 downstream으로 흐르는 비용을 줄이며, downstream DQS 기준으로 upstream에서 품질을 검증한다.
- Reconciliation은 primary control로 부적합 → pre-use validation을 primary로, reconciliation을 secondary verification으로 전환하는 것이 best practice.
- 모델은 template로 활용해 자사 function·data flow·employee cost에 맞춰 정교화 가능.
- 다음 챕터(7장)는 data를 asset으로 정의·curation하는 data governance를 다루며, data definition·architecture·management best practice와 data quality의 정렬을 설명한다.
