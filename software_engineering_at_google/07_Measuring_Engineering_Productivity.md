# 07. Measuring Engineering Productivity

## 챕터 개요 (3줄 요약)
- 조직 규모가 선형으로 커지면 소통 비용은 제곱으로 증가하므로, 개인의 생산성을 높여야 사업 범위를 확장할 수 있다.
- 측정에는 비용이 들기에 먼저 "측정할 가치가 있는가(triage)"를 묻고, GSM(Goal/Signal/Metric) 프레임워크와 QUANTS 5요소로 의미 있는 지표를 설계한다.
- AI 시대에 생산성이라는 인간적 요소를 데이터 기반으로 측정·개선하는 전문 역량은, 시니어가 조직 효율을 체계적으로 끌어올리는 레버리지가 된다.

---

## 1. Why Measure Productivity? (왜 측정하는가)
> 소통 비용은 인원에 비례해 선형으로 늘지 않으므로, 개인 생산성 향상이 확장의 열쇠다.

- 단, 개선 사이클 자체도 인적 자원을 소모하므로 "효율적으로" 측정·개선해야 한다.
- Google은 SW 엔지니어링 연구자뿐 아니라 인지심리학·행동경제학 등 사회과학자를 포함한 전담 연구팀을 구성했다.
- 사례: C++/Java 팀의 Readability 프로세스가 비용 대비 가치가 있는가? (autoformatter·linter 등장 후 "신고식(hazing)"이 아니냐는 의문).

---

## 2. Triage: Is It Worth Measuring? (측정할 가치가 있는가)
> 측정은 비싸고 엔지니어 행동을 바꿀 수 있으므로, 구체적 질문과 행동 가능성을 먼저 확인한다.

- 핵심 질문들: 어떤 결과를 기대하며 왜인가? 긍정 결과면 어떤 행동을? 부정 결과면 적절한 행동을 취할 것인가? 누가 언제 결정하는가?
- 측정하지 말아야 할 경우: 지금 프로세스를 못 바꿈, 곧 무효화될 결과, 결정권자의 확고한 신념, 허영 지표(vanity metrics), 부정확한 지표뿐일 때.
- 핵심: 측정의 성공은 가설 증명이 아니라 "결정권자에게 결정에 필요한 데이터를 주는 것" — 데이터가 쓰이지 않으면 실패다.

---

## 3. GSM Framework & QUANTS (지표 선택)
> Goal -> Signal -> Metric 순서로 만들어 가로등 효과(streetlight effect)와 지표 편향·확산을 막는다.

- Goal(목표): 측정 방법 언급 없는 원하는 결과. Signal(신호): 목표 달성을 알 수 있는 방법(측정 불가일 수도). Metric(지표): 신호의 측정 가능한 대리(proxy).
- LOC(Lines of Code)는 좋은 지표가 아니다 (Dijkstra: 코드 라인은 "생산"이 아니라 "지출").
- 추적성(traceability) 유지: 모든 지표는 신호로, 신호는 목표로 거슬러 올라가야 한다.

```
QUANTS - five components of productivity (in trade-off)
Q - Quality of the code
U - Attention from engineers (focus/flow)
A - iNtellectual complexity (cognitive load)
N -   (part of mnemonic)
T - Tempo and velocity
S - Satisfaction
* Improving one (e.g. velocity) can harm another (e.g. quality)
* Extreme example: "remove code reviews -> velocity fast, quality crashes"
```

---

## 4. Validating Metrics with Data (지표 검증)
> 정량 지표는 규모·신뢰를, 정성 지표는 맥락·서사를 제공하며 둘은 상호 검증한다.

- 사례: median build latency 지표가 자동 빌드를 포함해 "전형적 경험"을 왜곡 → experience sampling 연구로 발견·보정.
- 정량과 정성이 불일치하면, 대개 정량 지표가 기대 결과를 못 잡은 것이었다(Google 경험칙).
- Readability 측정은 3개 소스 결합: readability 직후 설문, 분기별 대규모 설문, 개발자 도구 로그(리뷰/제출 시간).
- 경고: 이런 지표를 개인 평가에 쓰면 게이밍(gaming)되어 무용지물 — 반드시 집계(aggregate) 효과로만 측정.

---

## 5. Taking Action & Tracking (행동과 추적)
> 권고는 "도구 주도(tool-driven)"여야 한다 — 적절한 데이터와 도구가 있으면 엔지니어는 올바른 트레이드오프를 한다.

- Readability 결과: 전반적으로 가치 있음 — 만족·학습 + 로그상 더 빠른 리뷰·제출(리뷰어 수 감소를 감안해도).
- 식별된 개선점을 언어팀이 도구·프로세스에 반영해 더 빠르고 투명하게 개선.

---

## Summary (핵심 정리)
- 측정 전 "결과가 실행 가능한가"를 먼저 묻고, 실행 불가면 측정하지 마라.
- GSM 프레임워크로 목표→신호→지표를 추적 가능하게 설계하고, QUANTS로 생산성의 모든 측면(특히 트레이드오프)을 커버하라.
- 정성 지표도 지표이며, 개발자 워크플로·인센티브에 내장된 권고가 실제 변화를 만든다 — AI 시대 조직 효율의 체계적 레버리지.
