# 10. Enterprise Data Management

## 챕터 개요 (3줄 요약)

- EDM은 data governance·contemporary architecture·MDM·data quality engineering·management operation의 best practice를 결합해 data-quality-first 문화를 구현한다.
- 책 전체의 framework(shape of data·DQS·MDM·governance·project methodology)를 통합하는 마무리 장이다.
- "한 문제를 풀고 다음 문제로" — 점진적 접근과 전사 협업으로 데이터 품질을 engineering하라는 실행 가이드를 제시한다.

---

## 1. EDM의 가치

> EDM = 데이터·정보 asset을 정의·acquire·integrate·manage·use하는 조직 역량.

- 시너지 결과: 데이터 가치 인식↑, 효율 운영을 위한 데이터·quality 이해 embed, mastered data(사용 전 validated·approved), 관리의 consistency·efficiency·scale, validation·analytics 표준화, ownership·stewardship 역할 인식.
- EDM은 data-quality-first 문화를 촉진해 business 전략 목표 달성을 가능하게 함 (책의 모든 method·framework 포함).

---

## 2. 시작점과 실행 영역

> The Martian의 Mark Watney 인용처럼 — "그냥 시작하라. 한 문제를 풀고, 다음을, 또 다음을."

### Understanding Data Volumes
- 금융 데이터 대부분은 panel data(time series cross-section). 사용 중인 volume의 source·structure·data type·definition·statistics를 아는가?
- data dictionary·glossary·inventory로 catalog, datum·record count, time series date range 파악.

### Engineering Data Quality
- DQS는 datum 수준 quality tolerance를 측정하는 framework — science·math를 데이터의 물리적 유사 property(dimension)에 적용.
- 내부 Governance·Management·vendor로부터 quality scorecard를 받는가? 없으면 요청하라.

### Improving Efficiency
- "Time is money." 데이터 이슈 추적에 시간 낭비 = 비효율. DQS로 pre-use validation 설계·구현.
- poor-quality data의 심각한 결과: 잘못된 의사결정, compliance 실패, 잘못된 client report → 규제 findings·벌금·고객 상실. (수조 달러 타인 자산 관리 — 본인 401(k)도 포함.)

### Scaling Data Architectures and Pipelines
- legacy·monolithic·brittle 구조는 수정이 매우 어려움. 데이터 규모가 인간 capacity를 초과.
- MDM architecture + pre-use validation + exception 기반 anomaly detection = 대규모 assembly-line 생산과 동일 발상 (Coca-Cola 캔을 사람이 일일이 검사하지 않듯).
- approaching/out-of-tolerance metric(anomaly)에 가장 주목하면 critical volume의 모든 datum을 systematic·quantitative하게 검증 가능.

### Achieving a Data-Quality-First Culture
- 데이터 품질은 steward나 Data Management만의 책임이 아닌 **모두의 책임**. data owner·steward·business·technical function의 partnership과 공유된 commitment 필요.

---

## Summary (핵심 정리)

- EDM은 governance(7장)·MDM(8장)·project methodology(9장)를 통합하는 roadmap·tool·template을 제공한다.
- 이제 data volume의 shape·structure 정의, pre-use validation engineering, metric으로 quality 측정, anomaly 식별, 역할·책임 정의의 도구를 모두 갖춤.
- 마지막 메시지: "다음 챕터는 당신의 것이며, 첫 문제를 선택하는 것에서 여정이 시작된다. Data quality engineering begins with you."
