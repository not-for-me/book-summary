# 03. Data Quality Specifications

## 챕터 개요 (3줄 요약)

- DQS는 downstream consumer의 tolerance 요구를 정의하며, 각 datum을 valid(V)·suspect(S)·invalid(IV) 상태로 판정한다.
- 8개 dimension(completeness·timeliness·accuracy·precision·conformity·congruence·collection·cohesion)별 tolerance와 shorthand 표기법, business impact(H/M/L)을 정의한다.
- Congruence는 prior value·average·standard deviation/z-score 세 기법으로 outlier·anomaly를 정량 탐지하며, z-score가 가장 정밀하다.

---

## 1. Manufacturing Controls & DQS Overview

> 제조업이 control spec을 쓰듯 금융업은 DQS로 data quality validation을 engineering하고 anomaly를 식별한다.

- DQS = consumer의 data tolerance 요구 + shape·quality 기대. validation app/platform에 embed되는 data quality rule.
- Pre-use validation이 DQS 미충족 데이터의 downstream 오염을 차단. anomaly/outlier detection 기법 사용.
- **Business impact level**:
  - **H (High)**: 사용 process가 failure 상태. 즉시 대응·remediation 필요.
  - **M (Medium)**: 운영은 가능하나 impaired. triage·remediation·reprocessing 필요(critical 아님).
  - **L (Low)**: 영향 경미, 운영 가능. reprocessing 불필요할 수 있음.

### 기본 용어

- **Datum**: data record 내 data element의 distinct instance (다른 데이터와 관계 가짐).
- **Data collection**: 모든 record가 존재해야 complete (index, benchmark, ETF, portfolio holdings).

---

## 2. Data Quality Tolerances (공통 코드)

> Tolerance는 항상 consumer/application의 quality 기대에 기반한다.

- **M** = Mandatory (datum 필수 존재)
- **O** = Optional (존재 여부 무관)
- **V** = Valid / within tolerance
- **S** = Suspect / approaching out of tolerance
- **IV** = Invalid / out of tolerance

### Shorthand 표기

```text
data element : dimension = M/O, V조건, S조건, [impact], IV조건, impact

예시:
close price: completeness = M, IV ≥ 1, H
available prices: completeness = O, V ≥ 80%, S ≥ 60% and < 80%, M, IV < 60%, H
```

---

## 3. Completeness

> 데이터 존재/부재. 가장 기본 test. null·empty string은 nothing이며 차원 측정 불가(개수만 count 가능). 단, 0이나 공백 문자열은 empty가 아님.

- 예: portfolio market value = position quantity × close price. close price는 M, 하나라도 비면 IV, impact H.

---

## 4. Timeliness

> 데이터의 temporality를 consumer DQS의 timing 기대 대비 평가. 금융 데이터 대부분이 time series.

- 10년 전 datum이 historical 분석엔 valid지만, 최근(예: past week)만 valid한 DQS엔 invalid.
- V/S/IV는 valid reference temporal range 기준. specific date에는 S 미적용.
- 예: `analyst estimate date: timeliness = O, V ≤ 60 days, 60 days > S ≤ 90 days, L, IV > 90 days, L`

---

## 5. Accuracy

> datum의 correctness. authoritative source 비교(direct) 또는 triangulation(indirect)으로 검증.

- **Authoritative source comparison**: 검증된 source/control data와 cross-reference (가장 robust한 direct 방식). 예: country/currency/classification code, position quantity vs source accounting system.
- **Triangulation**: 다른 data·calculation·시점으로 간접 검증 (indirect). 정확한 datum을 특정하긴 약함.
- 관계 기호: `+`(함께 사용), `=`/`≠`(equivalent 여부), `~`/`!~`(valid relationship 여부).
- 예(triangulation): `market value: accuracy—triangulation = V, Σ holdings market value = market value, IV, ≠, H`
- 예(APC/USD 오류): ticker symbol과 local currency 불일치로 price 오류 간접 검출.

> NOTE: authoritative comparison은 robust하지만, triangulation은 다른 데이터도 틀릴 수 있어 정확한 오류 datum 특정에는 약함.

---

## 6. Precision

> 숫자의 scale(소수 자릿수). 금융 계산에서 매우 중요 — 부정확한 rate/valuation은 자산 손실·규제 벌금으로 이어짐.

- 예: \$200,000를 cross rate 1.05 vs 1.059047로 환전 시 \$1,627.16 차이.
- 예: `cross rate: precision—decimal = V = 6, IV < 6, H` / `prices: precision—decimal = V ≥ 5, S = 4, M, IV ≤ 3, H`

> WARNING: Parquet·SQL column·Python/R data type 전반에서 precision 일관성을 보장하고, 의도치 않은 rounding/truncation을 경계해야 함.

---

## 7. Conformity

> 특정 format/standard 준수 여부 (binary: V/IV). 예: ISO country/currency code, GICS code, date format.

- 예: `GICS code: conformity = V, = format, IV, ≠ format, H`
- conformity는 format만 검증 → 올바른 code인지는 accuracy 검증, 누락은 completeness 검증 별도 필요.

---

## 8. Congruence (anomaly/outlier 탐지)

> 특정 datum이 동일 data element의 과거 관측치와 얼마나 유사한지(autocorrelated data). accuracy는 검증하지 않고 similarity만 정량 측정.

### 3가지 validation 기법

- **Prior value comparison**: 직전 시점 값과 비교. alpha는 string match, numeric은 % 차이.
  - 예(numeric): `close price: prior value congruence = V < 10%, S ≥ 10% and ≤ 20%, L, IV > 20%, H`
  - 공식: `% diff = |close(t) - close(t-1)| / ((close(t)+close(t-1))/2) × 100`
- **Comparison to average (mean)**: numeric 전용. 과거 평균과 비교. mean은 outlier에 영향받으므로 validated history 사용 권장.
  - 예: `close price: average congruence = V < 2%, S ≥ 2% and ≤ 3%, L, IV > 3%, H`
- **Standard deviation & z-score**: numeric 전용. 가장 정밀. date range를 windowing function으로 써서 최근 값끼리 비교(시장 상황 localize).
  - z-score = (datum - mean) / std_dev (지정 기간 기준)
  - 예: `close price: z-score congruence = range 5 business days, V ≤ 2, S > 2 and < 4, H, IV ≥ 4, H`

```text
Tolerance 설정 trade-off:
  narrow tolerance → false positive 多 (valid를 suspect/invalid로)
  wide tolerance   → 실제 invalid 미탐지
  → 데이터 특성(변동성)에 맞춰 폭 조정
```

---

## 9. Collection

> portfolio·index·ETF 같은 특수 data volume. 모든 구성 record가 존재·식별 가능해야 valid (V/IV binary).

- 예: S&P 500은 500개 constituent 모두 있어야 valid. 485개면 IV.
- collection check는 존재 여부만 확인 → datum 값 검증은 다른 dimension으로.

---

## 10. Cohesion

> datum 간 관계(주로 logical record). primary/foreign key로 dataset 간 join 가능 여부 확인 (V/IV).

- 예: close price \$130.10(06/15/22)이 AAPL의 것임을 ticker로 연결.
- 식별자: CUSIP, SEDOL, ISIN, account number 등.

> CAUTION: CUSIP/SEDOL/ISIN은 시간에 따라 unique 보장 안 됨(거래소별 중복, delisting·corporate action으로 재사용). historical time series엔 system-generated identifier 권장.

### DUPLICATES

- 중복은 dimension이 아니며, process/logic 오류로 발생. 중복 data는 corrupted로 간주하고 duplication check 필요. MDM·key 구조가 예방.

---

## Summary (핵심 정리)

- 8개 dimension은 단순 수학·텍스트 비교로 측정 가능하며, 생성된 metric을 consumer DQS와 대조한다.
- Pre-use validation을 data pipeline에 engineering·통합하면 consumer 제공 전 데이터 품질을 보장할 수 있다.
- 다음 챕터(4장)는 DQS framework를 실제 data volume에 적용해 tolerance 정의·metric 생성을 보여주는 상세 예제를 다룬다.
