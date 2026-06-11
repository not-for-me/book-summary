# 01. The Impact of Data Quality on Organizations

## 챕터 개요 (3줄 요약)

- Bad data는 "데이터가 더 이상 business objective를 지원하지 못하는 지점"으로 정의되며, perfect data를 목표하는 것은 경제적으로 비합리적이다.
- Poor data quality는 process/efficiency, reporting/analytics, compliance, data differentiation 4개 영역에 invisible drain으로 작용한다.
- Bad data의 주요 원인은 data culture 부재, process speed > governance 우선, M&A 데이터 마이그레이션이다.

---

## 1. What is Bad Data?

> Bad data = 데이터가 business objective 달성을 막는 지점. perfect data가 아니라 fit-for-purpose threshold를 정의해야 한다.

- Bad data가 유발하는 문제 유형:
  - process를 on-time(SLA)·within budget·appropriate outcome로 완료 못 하게 함
  - 의사결정에 필요한 정보가 missing/delayed/incorrect
  - compliance risk (regulator 제출 누락·지연, GDPR 등 privacy law 위반)
  - data product로서 차별화 실패
- **Bad vs perfect data**: 마지막 1% 개선의 효익은 marginal. fit-for-purpose threshold를 highly specific하게 정의해야 함.

### Tax ID 예시 (threshold 설정)

- 영국 VAT 등록은 revenue £85,000 미만 시 optional → 필드를 mandatory로 못 만듦.
- segmentation 권장: 대기업 threshold 95%, 소기업 60%.
- threshold가 부정확하면 false negative 발생 → 동료가 reporting 신뢰 상실 ("boy who cried wolf").

---

## 2. Impact of Bad Data Quality

> Gartner(2018): poor data quality는 연평균 \$11.8M 손실, 그러나 57%는 비용을 모름 ("unknown unknowns").

- 비용 정량화는 본질적으로 어려움: 수동 보정 인건비, 이탈 고객의 missed revenue 등 holistic 측정 거의 불가능 → executive 승인의 deal-breaker가 되곤 함.

### 사례 교훈 (e-invoicing 프로젝트)

- 정량적 business case가 약한 data quality 이니셔티브는 거부되었으나, 6개월 후 e-invoicing 프로젝트가 supplier master data(email·VAT) 품질 미달로 3개월 지연.
- **교훈 1**: small하게 시작 — 이슈가 명확한 한 가지 data type(customer/product) 선택, 적은 budget 요청, delivered value 입증.
- **교훈 2**: stakeholder에게 "왜 효익 정량화가 어려운지"를 사전 개별 설명해 mindset shift 유도.

---

## 3. Impacts of Bad Data In Depth (4개 영역)

### Process & Efficiency
- SLA miss → employee 불만, 관계 시작 지연, 계약 deadline 미준수, 경쟁사에 기회 상실.
- 팀이 잘못 sizing됨 → contractor 증원(30-50% 비용↑) 또는 기존 팀 과부하(stress·attrition).
- 증원 불가 시 → output 품질 저하, 일부만 우선처리(대형 고객 우선) → 평판 손상.
- NOTE: employee survey "processes allow me to be effective"가 항상 최악 응답, 그 중 ~30%가 data quality 관련.

### Reporting & Analytics
- 요약 데이터일수록 end user가 quality 이슈 탐지 어려움 (senior일수록 더 어려움).
- 예: 2010 UK 교통사고 차트 — 11월 1주 데이터 누락으로 "최고의 달"처럼 보이나 실제론 최악의 달 → 잘못된 의사결정 유발.

### Compliance
- 규제 산업 밖에서도 영향: 외부 audit이 control 기반 → control 미작동 시 substantive audit으로 회귀(audit fee↑, 최악엔 qualified opinion = 투자자 red flag).
- Financial Services: 규제기관이 taxonomy 데이터 요구, 품질 미달 시 oversight 강화·자본 추가 적립(BCBS 239 — "Principles for effective risk data aggregation and risk reporting").
- Pharma: FDA·MHRA 무예고 inspection, deviation 데이터 품질 불량 시 site 폐쇄 가능 → "data quality는 생사의 문제."

### Data Differentiation
- data가 product 자체이거나 차별화된 customer experience(예: 추천 알고리즘)의 일부일 때 품질 정밀도 최고 요구.
- 한 product의 저품질이 전체 data product 매출에 영향. 사회적 공개·SNS 확산 시 평판 손상.
- 사례: master data analyst 실수로 전체 가격 export를 한 고객에게 전송 → data breach, 직원 해고·고객 관계 파탄. (개인정보였다면 GDPR 위반.)

---

## 4. Causes of Bad Data

> 어떤 조직도 의도적으로 bad data를 계획하지 않는다. 주요 원인 3가지:

### Lack of a Data Culture
- 성공 조직은 holistic data culture·data literacy 교육. "데이터를 asset로 취급한다"는 말은 흔히 피상적.
- Doug Laney《Infonomics》: 물리적 asset·재무에는 엄격한 관리가 있으나 information asset에는 없음. 무형자산(특허·goodwill)은 asset register에 등재·감가하면서 왜 데이터는 안 하는가?

### Process Speed over Governance
- process 속도 vs data governance는 항상 긴장 관계. governance를 red tape로 보고 rule 제거 → data & process breakdown.

```text
Process speed vs Data quality (Figure 1.3 모델)
  box 1: fast, few rules    → poor quality (unsustainable)
  box 2: data & process breakdown (입력은 빠르나 후속 process 붕괴)
  box 3: rules 있으나 비효율 (신규 process 시작점)
  box 4: technology로 validation·orchestration (개선)
  box 5: 외부 DB lookup(D&B) 등 enrichment (best)
  box 6: governance 과복잡 → process owner가 box 1로 회귀
```

### Mergers and Acquisitions
- 공격적 timeline의 데이터 마이그레이션 문제: 두 소스 간 미de-duplication, as-is 마이그레이션(신규 시스템 요구 미반영), 저품질 데이터 개선 시간 부족.
- 합병 후 harmonization 투자는 있으나, 마이그레이션 문제 해결용 BAU budget은 거의 없음.

---

## Summary (핵심 정리)

- Bad data가 조직과 구성원에 미치는 영향, 그리고 조직이 의도치 않게 저품질 상태에 이르는 경로를 살펴봤다.
- 핵심: fit-for-purpose threshold를 specific하게 정의하고, small하게 시작해 tangible value를 입증하며, executive support를 확보하는 것.
- 다음 챕터(2장)는 data quality 개선 전 이해해야 할 핵심 개념과, 책 전체를 구조화하는 data quality 관리 approach를 제시한다.
