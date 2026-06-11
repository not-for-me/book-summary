# 06. Implementing Notifications While Avoiding Alert Fatigue

## 챕터 개요 (3줄 요약)
- notification은 monitoring과 실제 개선 행동을 잇는 핵심 고리로, triage·routing·resolution·documentation 각 단계를 지원하도록 설계해야 한다.
- alert fatigue는 monitoring 실패의 가장 큰 원인이므로, 적절한 audience·channel·timing과 다양한 suppression 기법으로 noise를 줄여야 한다.
- check 순서 스케줄링, ML clustering, priority level, 자동 root cause analysis 등으로 alert 가치를 높일 수 있다.

---

## 1. How Notifications Facilitate Data Issue Response
> monitoring은 productive action으로 이어질 때만 data quality를 개선하며, notification이 그 link다.

- **Triage**: issue가 우려스러운지 판단. monitor는 변화가 business에 중요한지 agnostic하므로, triage를 쉽게 하고 저우선 alert는 mute.
- **Routing**: 누구에게 알릴지 결정. 조직 구조에 의존(infra, product eng, data partnerships, marketing ops, data eng, analytics eng 등 issue별 담당 팀 매핑).
- **Resolution**: shock(수정)·scar(backfill)를 고치는 실제 작업. Linear/GitHub/Jira 등 별도 추적.
- **Documentation**: 종료 시 조치 내용 기록 — 유사 issue 재발 대비, 잘못된 fix 추적·복구.

### 1.1 Taking Action Without Notifications
> 제한적 시나리오에선 notification 없이도 조치 가능하다.

- orchestration(Airflow/dbt)에 check 통합 — staging에서 통과 시 production publish, 실패 시 자동 cleanup(중복 제거)이나 publish 차단(틀린 data보다 늦은 data가 나을 때). freshness/volume은 retry로 자체 해결되기도.
- 단 진짜 bad data를 mutate로 덮는 건 대개 나쁨 — issue를 숨기고 bias 주입. API·dashboard 수동 검토는 정의상 scale 안 됨.

---

## 2. Anatomy of a Notification
> 좋은 notification은 triage·route·resolve·document를 최대한 쉽게 만들어야 한다.

- **Visualization**: notification당 정확히 하나, check·issue별로 type 달라짐. 현재 issue + 맥락·history 제공하되 너무 복잡하지 않게.
- **Actions**: 즉시 행동 가능하게 3개 button — **View details**(플랫폼에서 조사), **Edit check**(config 수정), **Triage**(Acknowledge/Resolve/File a ticket).
- **Text Description**: **Title**(사용자 정의 의도), **Summary**(자동 생성, issue 성격·심각도 정량화), **History**(과거 발생 횟수 — 수백 일 침묵 후 발생 vs 4회 연속).
- **Who Created/Last Edited**: alert 수신자가 질문할 대상 파악에 유용.

---

## 3. Delivering Notifications
> who(누가)·how(어떤 channel)·when(즉시/요약) 세 차원으로 전달을 결정한다.

### 3.1 Notification Audience
> 누구에게 알릴지가 alert fatigue 감소의 핵심이다.

- **ownership과 context** 중심 — 수신자는 해당 table의 quality에 높은 ownership을 가져야 하고 필요한 context를 제공해야 함. 그 후 audience를 **최소화**(notification은 방해 요소). 관심만 있는 사람은 report·dashboard로.
- 흔한 전략: alert를 table별로 묶고, table을 관심 user **domain**별로 묶음(marketing, finance, growth, operations 등). issue 유형으로도 결정(data eng은 freshness/volume, analytics는 metric).
- **철칙: 한 alert는 오직 하나의 audience에게** — 그래야 ownership 유지·단일 대화. 아니면 low ownership(서로 미룸)이나 duplicate effort(중복 작업).

### 3.2 Notification Channels
> audience·issue 유형별로 적합한 channel이 다르다.

- **Email**(비동기, 주간 summary), **Real-time**(Slack/Teams — emoji reaction triage, thread 대화), **PagerDuty/Opsgenie**(가장 중요한 드문 check, on-call 호출), **Ticketing**(Jira/ServiceNow — 단 사람이 중간 triage 권장), **Webhooks**(임의 system 라우팅).

### 3.3 Notification Timing
> check 완료 즉시 개별 전송 vs 그룹 완료 후 summary 전송 — 전자를 강력 권장.

- 즉시 개별 전송 이유: 빠른 대응(check별 runtime 차이 큼), "all green" summary는 무행동 소비로 alert fatigue 유발, issue별 개별 thread로 대화 분리.

---

## 4. Avoiding Alert Fatigue
> noisy notification으로 user를 압도하지 않는 것이 monitoring 설정의 가장 까다로운 문제다.

### 4.1 Scheduling Checks in the Right Order
> check 스케줄링은 false positive의 가장 큰 원천 중 하나다.

- naive 접근(데이터 적재 직후 전체 실행 / 고정 시각 실행) 모두 **incomplete data**에서 실행될 위험 — 지연·부분 도착 시 cross-table·aggregation rule이 거의 확실히 실패. incomplete data는 거의 항상 biased.
- 해법: 먼저 **data freshness**(partition 내 record 존재, cadence 학습·적응)와 **data volume**(time series로 예상 분포) check를 15분마다 실행해 **gate** 역할 → 통과 후에만 deep data quality check 실행.

### 4.2 Clustering Alerts Using Machine Learning
> 같은 issue에서 나온 중복 alert를 clustering·dedup하는 것이 중요하다.

- 중복 원인: freshness/volume이 downstream으로 cascade, 밀접 관련 column 다수, exogenous effect.
- table 내: record를 good/bad로 분류 후 여러 issue 간 bad/good record 상관을 계산해 같은 set인지 판정(Ch4의 SHAP 값 활용). table 간: **data lineage**로 upstream 실패 확인해 clustering.

### 4.3 Suppressing Notifications
> user에게 도달하기 전 notification을 억제하는 여러 방법이 있다.

- **Priority level**: **High**(매번 실패 시 + 복구 첫 통과 시 alert), **Normal**(연속 실패 첫 3회 후 억제, 기본값), **Low**(절대 alert 안 함, debugging·문서화용).
- **Continuous retraining**: 매일 재학습으로 "known issue"를 new normal로 인식해 자동 억제(day1 issue는 day2부터 억제).
- **Narrowing scope**: 중요 table만 ML 모니터링(Table 3-1), deprecated legacy column 제외, **SQL query log**("heat in the data")를 파싱해 가장 많이 쓰이는 table·column으로 범위 축소.
- **Making check less sensitive**: 허용 범위 확대, 신규 bad record만 alert, time series check의 **confidence interval** 조정(80%면 ~5일마다, 99%면 ~100일마다 alert).
- **What not to suppress: expected changes**: 예상된 변화도 억제하지 말 것 권장 — expected change 목록 유지가 어렵고 부정확하면 trust 훼손. 차라리 alert가 변화 발생을 확인·문서화·context 제공하게.

---

## 5. Automating the Root Cause Analysis
> resolution 전에 문제 위치를 찾아야 하며, 전통적으로 비싼 이 작업을 자동화할 수 있다.

- Ch4의 unsupervised ML로 good/bad row 차이를 설명할 수도 있으나, user는 주로 "어떤 segment가 bad data를 가장 잘 설명하는가"를 원함 → 더 단순·저비용 접근: data sampling 후 segment별(`WHERE column = x`) 독립 분석.
- 핵심 계산: segment 내 bad row 비율 `D/(C+D)` vs good row 비율 `B/(A+B)` 비교 — bad가 과대표현·good이 과소표현인 segment를 탐색.
- 4 분류: 비율 동일(무관), 모든 bad + 많은 good(방향성만), 대부분 bad지만 bad 누락(불충분), **모든 bad + 거의 그것만(root cause 발견 가능성 높음)**.
- 빠른 비율 계산이라 자동화해 likely root cause segment를 시각화(예: ticket price NULL의 culprit segment). good/bad sample을 탐색·export 제공 가능.

---

## Summary (핵심 정리)
- notification은 triage·routing·resolution·documentation을 지원하며, visualization·actions·text·작성자 정보로 구성된다.
- 전달은 who(ownership 기반 최소 audience, 한 alert 한 audience)·how(email/Slack/PagerDuty/ticket/webhook)·when(즉시 개별 전송)으로 결정한다.
- alert fatigue 방지: freshness/volume gate 스케줄링, ML·lineage clustering, priority level·retraining·scope 축소·sensitivity 조정 등 suppression.
- expected change는 억제하지 말고 확인·문서화에 활용한다.
- segment별 bad/good 비율 비교로 root cause analysis를 자동화한다.
