# 01. Thinking Like a Manufacturer

## 챕터 개요 (3줄 요약)

- 데이터를 raw/semifinished material로 보고, data management를 manufacturing production process처럼 다루는 사고방식을 제시한다.
- 제조업의 control specification·quality gate 개념을 데이터에 적용한 것이 DQS(Data Quality Specifications)이며, pre-use validation으로 품질을 보장한다.
- Lean manufacturing의 waste 제거 원칙을 데이터에 적용하면 DQS를 만족하지 못하는 poor-quality data를 줄이는 것으로 이어진다.

---

## 1. Operational Efficiency

> Operational efficiency는 business operation의 input 대비 output의 ratio이며, poor data quality는 금융업 비효율의 핵심 원인이다.

### 핵심 개념

- Operational efficiency = output-to-input ratio. input은 money·IP·employees, output은 products·revenue·customers 등.
- Poor data quality → 부정확한 business insight, 잘못된 financial analysis, 잘못된 investment decision으로 이어진다.
- 직원들이 데이터를 반복 검증하는 데 시간을 소모 → 효율 저하. regulator·auditor 대상 misrepresentation은 effectiveness에 심각한 영향.

### 규율 잡힌 data management로 가는 활동

- Manufacturing approach를 data quality management에 적용
- DQS(Data Quality Specifications) 구현
- Pre-use validation을 primary quality control로 적용
- Reconciliation을 post-use verification(secondary control)으로 전환
- 데이터 dimension별 quality 측정

---

## 2. Lessons from Lean Manufacturing

> Lean의 zero-waste 원칙을 데이터에 적용하면 DQS 미충족 데이터를 제거하는 것과 직결된다.

- Lean은 1980년대 후반 MIT IMVP의 Jim Womack 팀이 Toyota 생산방식을 묘사하며 사용한 용어.
- Lean organization은 개별 technology·asset·department 최적화에서 → value-generating process의 horizontal flow 최적화로 초점 전환.
- 목표: process 전반의 waste 제거 (less capital, less human effort, fewer defects, lower cost).
- 데이터 적용: DQS를 만족하지 못하는 poor-quality data 감소·제거.

---

## 3. Manufacturing Quality 사례 (Coca-Cola / DASANI®)

> 단순해 보이는 제품도 엄격한 quality specification과 다단계 validation을 거친다.

### Coca-Cola

- 완제품뿐 아니라 flavoring·water·container 같은 raw material까지 stringent quality control 적용.
- 생산 전 manufacturing line 점검, CO2 volume·water-to-syrup ratio 확인, 시작 30분 내 net contents 검사, torque check(캡 조임)·label 검사.

### DASANI® 정수 단계

- Activated carbon filtration (VOC·chlorine 흡착)
- Reverse osmosis (mineral·impurity 제거)
- UV disinfection (microorganism 제거)
- Remineralization (magnesium sulfate, potassium chloride, salt 추가로 일관된 맛)
- Ozonation (O3 최종 살균, 잔류 맛 없음)

---

## 4. Manufacturing Control Specifications

> 동일한 raw material도 consumer use case에 따라 전혀 다른 quality specification·tolerance를 가진다.

### Water Quality 3가지 use case

- **Ultrapure water**: 반도체 chip 생산용. H2O와 hydrogen/oxygen ion balance만 허용, 고순도.
- **Potable water**: 인체 음용 가능(safe to drink). pathogen·오염물질이 max safe tolerance 이하. 맛 좋음을 의미하지 않음.
- **Mineral water**: dissolved sodium·potassium·chloride·bicarbonate·sulfate·calcium·magnesium 조합으로 더 좋은 맛.

### NOTE — 핵심 원칙

- 동일 raw material을 여러 consumer가 여러 목적으로 사용 가능
- material의 quality specification은 consumer use case requirement가 정의
- use case 간 quality spec 차이가 극적으로 클 수 있음
- material이 quality spec을 만족해야 사용 가능

### Quality Control and Anomaly Detection

- 제조사는 sensor signal·tolerance measurement 기반 anomaly detection으로 예기치 못한 이벤트를 식별.
- 재료는 다음 단계로 가기 전 quality gate를 통과해야 함.

---

## Summary (핵심 정리)

- Manufacturing은 control specification으로 material의 물리적 특성·quality tolerance를 정의하고, data manufacturing은 동일하게 DQS로 통제한다.
- 같은 raw material/데이터도 consumer use case에 따라 quality spec이 달라지며, spec을 만족해야 production에 사용 가능하다.
- Pre-use validation이 DQS 미충족 데이터가 downstream ecosystem을 오염시키는 것을 방지한다.
- 다음 챕터(2장)는 the shape of data와 data dimension을 다루며, panel data·cross-section time series 등 금융 데이터를 dimension·tolerance 관점에서 측정하는 framework를 소개한다.
