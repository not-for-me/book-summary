# 02. The Shape of Data

## 챕터 개요 (3줄 요약)

- 데이터를 물리적 form을 가진 raw material처럼 사고하여, dimension 수준에서 quality를 평가하는 개념 framework를 제시한다.
- 금융 데이터의 shape을 data element·datum·time series·cross-section·panel data·data volume으로 구조화한다.
- 각 datum은 8개 data dimension(completeness·timeliness·accuracy·precision·conformity·congruence·collection·cohesion)으로 측정 가능하며, DQS 기반 tolerance test의 대상이 된다.

---

## 1. Data as Physical Asset

> 데이터를 firm의 중요한 physical asset으로 취급하면 industrial manufacturing 기법을 적용하기 쉬워진다.

- 데이터는 temporal dimensionality를 가짐:
  - **Dynamic data**: 시간에 따라 변함 (current analytics, trade list, portfolio position, transaction, cash flow, performance).
  - **Persistent data**: 거의 변하지 않음, history를 표현. 검증·확정된 historical data.
- Dynamic/persistent는 서로 다른 technology·technique로 관리·curation됨.
- 데이터는 technology 안에 존재 → 모든 data initiative는 technological component를 요구.

---

## 2. Data Shape Concept Model

> 데이터는 물리적 form이 없지만, shape을 정의하는 dimension에 측정 기법을 적용해 quality를 평가할 수 있다.

### Shape 구성 요소

- **Data element**: 고유 정의를 가진 가장 작은 named data item (예: close price, exchange rate, market value).
- **Datum**: data element의 단일 data point value (예: \$143.11).
- **Data universe**: 관련 item들의 named group (예: stock universe, currency universe).
- **Time series data**: 한 universe item·한 data element의 다중 시점 datum 집합. univariate = 1차원 array (longitudinal).
- **Cross-section data**: 한 시점·한 data element의 다중 item datum 집합. univariate = 1차원 array.
- **Panel data (time series cross-section)**: time series + cross-section 결합. multivariate = 3차원 matrix/tensor.
- **Data volume**: matrix 내 모든 datum의 총 count. 3차원 cube로 시각화. manufacturing의 raw material volume에 대응.

### TIP

- 데이터를 mass를 갖고 3차원 공간에 존재하는 physical volume처럼 시각화하는 것이 manufacturer 사고의 foundation.

```text
Panel data (3D volume)
        z = stock universe (Apple, Google, IBM)
        |
        |____ y = data element (open price, close price)
       /
      x = date

  → datum = intersection(date, price, stock)
```

### Data Volume 활용 예 — Portfolio market value

- 3개 data volume 필요: (1) portfolio security position 수량, (2) security instrument reference data, (3) 각 position의 price(close price).
- market value = position quantity × close price, 그리고 모든 security market value의 합(cash position 포함).
- 금융 데이터 대부분은 panel data (market data, fundamentals, securities/instruments, trades, positions, performance).
- 차이점: manufacturing raw material은 purity·repeatability로 균일하지만, 금융 data volume은 unique datum들로 구성되어 각 datum의 quality가 개별적으로 중요.

---

## 3. Data Dimensions and Attributes

> Data dimension은 consumer의 DQS에 따라 데이터의 shape·quality를 specific measurement로 이해하게 하는 실용 framework다.

### Data Attributes (metadata)

- **Type**: numeric, character, string, alphanumeric.
- **Meta**: datum을 설명하는 metadata (definition, classification, origination).
- **Owner**: 최고 수준 SME, DQS를 정의("fit for purpose"). 보통 데이터를 직접 관리하진 않음.
- **Steward**: 데이터 관리·curation 담당 운영팀 (acquisition, profiling, validation, remediation, metrics, preservation, dissemination). owner가 정의한 fit-for-purpose 데이터 전달 책임.
- **Security**: data masking·access control·confidentiality·integrity·storage.
- **Policy**: 데이터 사용 guideline·rule (열람·저장·파생·공유 허용 범위).

### 8개 Data Dimensions

- **Completeness**: 데이터 존재/부재 여부 (가장 기본 test). mandatory vs optional 판단.
- **Timeliness**: 시점/날짜, fresh vs stale.
- **Accuracy**: valid·correct 여부. authoritative source 비교 또는 triangulation.
- **Precision**: 소수 자릿수 또는 code value의 정밀도/scale.
- **Conformity**: 특정 format·standard 준수 (예: ISO Alpha-2/Alpha-3).
- **Congruence**: 다중 time period 간 유사/차이, autocorrelated data에 적용.
- **Collection**: named group의 모든 member datum 존재 (예: portfolio positions, index constituents).
- **Cohesion**: 한 datum과 다른 datum의 relationship (예: Position SEDOL = Security SEDOL).

### NOTE

- 대부분 dimension은 모든 데이터에 적용되나, collection은 collection의 member인 datum에만 적용.
- Dimension은 individual datum 수준의 intrinsic 특성. DQS framework로 각 dimension의 tolerance test를 정의해 downstream 제공 전 검증.

---

## Summary (핵심 정리)

- Dataset은 manufacturing의 raw material volume과 같고, data volume은 shape·dimension을 가진 다수의 datum으로 구성된다.
- 금융 analytical use case는 보통 여러 data volume을 요구하며, quality·integrity·cohesion이 data volume viability의 핵심 결정 요소다.
- 데이터는 사용 전 DQS에 따라 분석·측정·검증·certify되어야 한다.
- 다음 챕터(3장)는 DQS framework로 datum 수준 dimension의 data quality tolerance를 정의하는 방법을 다룬다.
