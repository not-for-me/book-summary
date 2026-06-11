# 04. DQS Model Example

## 챕터 개요 (3줄 요약)

- DQS framework를 manufacturing assembly line처럼 11개 business function 모델에 적용해, security master data volume에 각 dimension validation을 실제 수행한다.
- raw data → completeness·timeliness·accuracy·precision·conformity·congruence·collection·cohesion 순으로 검증하여 V/S/IV metric과 통계를 생성하고 anomaly를 remediation한다.
- Fit-for-purpose는 같은 데이터라도 consumer별 DQS·business impact가 달라짐을 보여주며, 6단계 level로 정리된다.

---

## 1. DQS Model 구조

> 데이터가 fit-for-purpose인지는 business function별 DQS tolerance에 따라 다르다.

- 11개 business function이 좌→우 assembly line으로 배치: Data Management(vendor 데이터 ingest·validation·remediation) → Research, Analytics, Portfolio Management, Trading, Compliance, Investment Operations, Business Development, Client Services, Performance Measurement, Marketing.
- 각 function box는 필요한 data volume과 V/IV/S 상태를 표시. security master data가 downstream으로 provisioning됨.
- 예: empty price는 일반 research엔 사용 가능하나 NAV 계산·performance 계산엔 not fit for purpose.

```text
Data Management (raw ingest, level 0)
   └─ apply DQS validations →
       Research / Analytics (medium quality)
       Portfolio Mgmt / Trading / Compliance (high quality)
       Investment Ops / Client Services / ... (high quality)
```

---

## 2. Dimension별 DQS 적용 (security master data volume, 25 records × 11 columns = 275 datum)

> 각 dimension validation은 shorthand DQS로 정의되고, 결과는 V/S/IV 개수 통계로 집계된다.

### Completeness
- 대부분 element는 M(mandatory): `Ticker: Completeness = M, IV ≥ 1, H`. Consensus Recommendation/Date는 O(optional), 누락 시 S.
- 결과: Valid 248, Invalid 22, Suspect 5.

### Timeliness
- Consensus Date에 적용: `V < 30 days, 30 ≤ S < 90, IV ≥ 90, L` (Processing Date 대비).
- 결과: V 15, IV 6, S 4.

### Accuracy
- Ticker/Issue Name/Exchange를 NYSE·NASDAQ 공식 listing(authoritative source)과 비교. Exchange로 어느 listing을 쓸지 선택.
- `Ticker: Accuracy—authoritative = V, Ticker = Ticker, IV, ≠, H`
- 결과: empty Exchange, malformed Issue Name 등 anomaly 검출.

### Precision
- Bid/Ask/Spread/PE: `Precision—decimal = V ≥ 1, S = 0, IV = negative, H` (decimal 없으면 suspect, 음수면 invalid).
- WARNING: decimal/float data type의 저장·계산에서 precision 손실·반올림 주의. DQS 요구 precision을 전 구조에서 일관 유지.

### Conformity
- Issue Name: `V = proper case, IV = empty/upper/lower, H`
- Market Cap Scale: `V = B or M, IV ≠ B or M, H`
- Consensus Recommendation: `V ≥ -3 and ≤ 3, S = empty, IV < -3 and > 3, L`

---

## 3. Congruence DQS (z-score 기반)

> stock별 historical 관측치 대비 z-score로 outlier를 탐지하며, 10 business day sample range를 사용한다.

- `Bid: Congruence z-score = range 10 business days, V ≤ 3, S > 3 and < 4, H, IV ≥ 4, H` (Ask·Spread 동일).
- z-score = (raw value - mean of prior 10 days) / std_dev.
- 예: IBM Bid z-score 15.75 → **IV**(invalid), Coca-Cola Bid z-score 3.31 → **S**(suspect), AAPL Bid 1.48 → V.
- 결과(15 datum): Valid 11, Invalid 2, Suspect 2. anomaly는 추가 inspection 필요.

---

## 4. Collection DQS (portfolio holdings)

> collection은 모든 record가 존재·식별 가능해야 valid. control total(record count, market value)로 검증.

- `Collection—record count = V, Raw = control, IV, ≠, H`
- `Collection—market value % diff = V, Raw < 3% control, IV, ≥ 3%, H`
- control total이 없으면 congruence prior value/z-score로 대체 가능.
- 예: account 987654의 record 누락 → record count 불일치, market value % diff 138.1% outlier로 IV 검출. (단, accuracy 외 모든 validation은 정확성을 보장하지 않고 anomaly만 식별.)

---

## 5. Cohesion DQS

> primary key ↔ foreign key 매칭으로 data volume 간 join 가능 여부를 검증한다.

- security master PK = (Processing Date, Ticker). portfolio holdings FK = (Processing Date, Ticker).
- `Cohesion = V, raw holdings (Date+Ticker) = security master (Date+Ticker), IV, ≠, H`
- 예: holdings에 ticker DELL(11/3/2015) 존재하나 security master에 없음 → cohesion IV. 추가 조사 필요.
- DB의 PK/FK 제약이 cohesion을 강제할 수 있으나, 다양한 legacy 기술 환경에선 별도 cohesion validation 구현이 필요.

---

## 6. Fit for Purpose (6 levels)

> 데이터 품질 요구와 poor-quality data의 business impact는 function별로 다르다.

```text
Level 0  Data Management              quality: Low     impact: None (raw ingest)
Level 1  Research                     quality: Medium  impact: waste resources/time
Level 2  Analytics                    quality: Medium  impact: poor analytics
Level 3  Portfolio Mgmt/Trading/Compliance  quality: High  impact: financial/regulatory/reputation
Level 4  Investment Operations        quality: High    impact: financial/regulatory/reputation
Level 5  BizDev/Client Svc/Marketing/Perf   quality: High  impact: financial/regulatory/reputation
```

- Medium impact → triage·remediation 필요(critical 아님). High impact → 즉시 대응.
- 같은 dimension의 DQS도 function마다 다를 수 있음(실무에선 보통 다름). research/data science의 탐색용 데이터는 production보다 tolerance가 낮을 수 있음.

---

## Summary (핵심 정리)

- Dimension은 일반성 스펙트럼을 가짐: **General** = completeness(모든 데이터), **Specific** = accuracy·collection·cohesion·congruence·timeliness·conformity·precision(데이터 성격에 종속).
- DQS framework 적용은 대량의 data quality metric을 생성하며, 이를 통해 데이터의 shape을 V/S/IV tolerance로 정량 측정할 수 있다.
- 다음 챕터(5장)는 이 metric들을 빠르게 이해하기 위한 data quality visualization(valid/suspect/invalid의 spectrum·density 매핑)을 소개한다.
