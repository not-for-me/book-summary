# 02. Data Quality Monitoring Strategies and the Role of Automation

## 챕터 개요 (3줄 요약)
- 성공적인 data quality monitoring은 detect, alert, resolve, scale 네 dimension을 모두 충족해야 하며, alert fatigue(false positive 과다)는 도입 실패의 핵심 원인이다.
- manual detection, rule-based testing, metrics monitoring 등 traditional 접근은 각자 가치가 있지만 enterprise scale에서 한계가 명확하다.
- 저자는 data observability + rule-based testing + metrics monitoring + unsupervised ML의 four-pillar approach를 권장한다.

---

## 1. Monitoring Requirements
> 성공적 monitoring은 detect(table/column/row 단위 모든 issue 탐지), alert(적시·정확), resolve(빠른 해결), scale(enterprise 전반) 네 dimension을 충족해야 한다.

- **Alert fatigue**: false negative(실제 issue 미알림)와 false positive(불필요 알림) 모두 문제. false positive가 많으면 사용자가 notification을 무시·비활성화 → "platform that cried wolf". monitoring 도입 실패의 가장 흔한 원인.

---

## 2. Data Observability: Necessary, but Not Sufficient
> data observability는 data 내용이 아닌 table의 metadata(존재·schema·갱신·volume)를 monitoring한다.

- "이 table이 존재하는가? schema 변경은? 최근 갱신됐는가? volume이 기대치와 일치하는가?" 같은 질문에 답함.
- modern data warehouse의 API나 system view로 table 쿼리 없이 metadata 수집 가능. 예: Snowflake `INFORMATION_SCHEMA."TABLES"`에서 `ROW_COUNT, BYTES, LAST_ALTERED` 조회.
- "최근 갱신" 판단엔 time series model로 다음 update 기대 상한을 예측. 수만 개 table까지 확장 가능.
- 한계: data flow만 확인할 뿐 내용의 quality는 못 봄. (수압만 보고 식수 가능 여부는 안 보는 정수장과 같음.)

---

## 3. Traditional Approaches to Data Quality
> manual detection, rule-based testing, metrics monitoring 세 가지가 historically 가장 흔하지만 scale에서 한계가 있다.

### 3.1 Manual Data Quality Detection
> 사람이 직접 data를 훑어 issue를 찾는 방식 — 소규모엔 가능하나 scale엔 비효율적이고 subjective하다.

- spreadsheet의 약 90%에 오류가 있다는 연구도 있을 만큼, manual은 주관적이고 분석가마다 결론이 다름.
- 종종 "우연히" 발견됨: summary statistic 비교(고객 수 50% 불일치), visualization(최근 missing value 급증), 불가능한 결론(주 1,000% 성장, 1970-01-01 생일 집중).
- resolve-as-you-go 방식은 일부 조직에서 분석가 시간의 50%+를 issue 조사·우회에 소모시켜 효과·morale 저하.
- 단, human은 algorithm이 못하는 맥락 결합·판단이 가능하고, 분석 중 발견된 issue는 정의상 "important"(false positive 없음). 따라서 어떤 접근이든 manual profiling을 쉽게 돕도록 summary·visualization 제공이 바람직.

### 3.2 Rule-Based Testing
> software unit test처럼 data에 deterministic rule을 적용해 pass/fail로 판정하는 방식 (Great Expectations, dbt).

- rule은 **scope**(어떤 store·table·column·기간·row), **type**(unique, never NULL, 값 집합, schema, 다중 column 관계, 복잡 SQL join/subquery), **constraint**(50~100 같은 상수)로 구성.
- 장점: 저렴하고 mistake 없음, deterministic해 실패 이유 명확, false positive 불가(rule 자체가 틀리지 않는 한), **historical issue 탐지에 강함**(과거부터 잘못된 data도 SME가 first-principle로 규정), billions row에서 needle in haystack 탐지 가능.
- 단점: scope·constraint·type 오지정 여지 큼. enterprise 전체 cover는 Sisyphean — 10,000 table 예시에서 `1,000 table × 50 column × 5 rule = 250,000 rule`. data는 끊임없이 변해 unit test보다 maintain이 훨씬 어렵고 brittle.
- 결론: scalable하진 않지만 SME가 기대를 표현·강제하는 강력한 도구. 이상적 solution은 다양한 SME가 rule을 쉽게 생성·편집·분석하게 함.

### 3.3 Metrics Monitoring
> data 통계(NULL %, duplicate %, mean, min/max 등)에 threshold를 걸어 급변 시 alert하는 방식.

- 단점: 모든 column·segment·statistic을 cover하려면 surface area가 폭발. aggregate 수준이라 소수 record 영향 issue를 놓침. 변화 원인 record를 특정 못함. feature flag로 점진 rollout되는 gradual change는 threshold에 안 닿아 놓칠 수 있음.
- 장점: 특정 slice를 면밀히 볼 때 필수(예: Airbnb의 active listings, reactivation 등). 일정 비율 degrade가 유의하게 증가하는지 감시에 유효.
- **Time series metric monitoring**: 수동 threshold는 seasonal data에 brittle. exponential smoothing, ARIMA, Prophet, RNN 등으로 기대 범위를 학습해 seasonality·holiday·trend 보정. 대부분 권장하되, SLA 같은 명확한 한계엔 hand-coded range도 유효.

---

## 4. Automating Data Quality Monitoring with Unsupervised Machine Learning
> fraud detection·underwriting처럼 data quality도 unsupervised ML로 자동화해 scale·일관성을 확보할 수 있다.

- **supervised vs unsupervised**: supervised는 human label 필요 — data가 table·회사마다 천차만별이라 label 수집이 비현실적. unsupervised는 data 자체의 pattern을 학습해 초기 setup 없이 monitoring 시작·적응 가능.
- 탐지 가능 issue: NULL % 증가, 특정 segment(국가) 소실, column 분포 변화(credit score skew), **multi-column 관계 변화**(합이 같던 column들이 더 이상 안 맞음).
- "ML"이라며 실제론 time series metric만 돌리는 solution과 구분 — 본서의 unsupervised learning은 복잡한 dataset **전체**의 예상치 못한 변화 탐지를 의미.
- **핵심 강점**: table 전체의 column 간 correlation 구조를 이해(예: Taiwan credit card data의 phi-K correlation). column을 isolation으로 보면 상관된 monitor가 동시에 수십 개 alert를 내고, age-limit_balance correlation 변화 같은 contextual signal을 놓침. ML은 correlated issue를 하나로 clustering하고 unknown unknowns까지 탐지.

### 4.1 An Analogy: Lane Departure Warnings
> 운전의 lane 유지처럼, ML은 hard-coded rule로 잡기 어려운 방대한 context를 반영한다.

- **manual**: 운전자가 직접 차선 주시 — 집중 요구, 사고 잔존.
- **rules/metrics**: 노란 pixel 비율 threshold — 차선이 노랑·실선이 아니거나 좌회전·차선변경 시 false positive 폭주 → 사용자가 alert 비활성화.
- **ML**: turn signal·obstacle·충돌 위험 등 context를 반영해 똑똑하게 알림 → noise 감소, 신뢰·안전 향상.

### 4.2 The Limits of Automation
> 잘못 설계된 automation은 no automation보다 나쁠 수 있고, 일정 수준 이상은 diminishing returns다.

- unsupervised ML의 한계: ① sampling이라 needle in haystack은 rule만큼 못 잡음, ② **new change**만 보므로 historically 항상 틀린 data(missing을 0으로 코딩)는 못 잡음, ③ 모든 column·row를 동등 취급해 특정 critical slice 주목도는 metric보다 낮음.
- **rule/metric 자동 생성**: 가능하나 비용·brittleness·false positive 큼. rule 자동 생성은 모든 historical record 평가가 필요해 비싸고, 소규모 table엔 우연히 통과해 brittle. 정작 중요한 rule은 SQL custom 필요. → 사용자가 자발적으로 rule 추가하게 하는 편이 낫고(중요한 rule만 생성, 실패 시 학습 기회), metric은 out-of-box 일부 자동화하되 custom metric 정의를 반드시 허용.

---

## 5. A Four-Pillar Approach to Data Quality Monitoring
> 저자 권장: data observability + unsupervised ML(기본 coverage) + low-code validation rule + time series metric monitoring을 결합한 four-pillar.

- 가능한 접근 스펙트럼: do nothing → observability만 → 일부 handcrafted rule/metric → 전체 handcrafted(비싸고 noisy) → rule 자동화(brittle) → metric 자동화(noisy) → unsupervised만(critical data 주목도 부족) → **four-pillar(권장)**.
- four-pillar는 detect·alert·resolve·scale을 균형 있게 달성: observability는 전 warehouse 저비용 적용, unsupervised ML이 obvious issue·unknown unknowns 기본 cover, SME는 중요 data에 low-code rule·time series metric으로 보강.
- 이 조합으로 false positive·alert fatigue 최소화하며 실제 risk를 높게 cover — 분석가 군단 불필요.

---

## Summary (핵심 정리)
- monitoring은 detect, alert, resolve, scale 4 dimension을 충족해야 하고, alert fatigue 방지가 성패를 가른다.
- data observability는 필수지만 metadata만 보므로 불충분하다.
- traditional 3접근(manual, rule-based, metrics)은 각자 강점이 있으나 enterprise scale에선 비싸거나 brittle하거나 noisy하다.
- unsupervised ML은 table 전체의 correlation 구조를 학습해 unknown unknowns·correlated issue를 cluster로 탐지하나, needle/historical/critical-slice엔 약하다.
- 권장 해법은 observability + unsupervised ML + rule + metric의 four-pillar approach다.
