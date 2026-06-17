# 14. Doing the Right Thing

> Keywords: ethics/responsibility, predictive analytics, algorithmic prison, bias/discrimination, protected traits/proxy, accountability/transparency, feedback loop/systems thinking, privacy/tracking, surveillance, consent/freedom of choice, data as asset/toxic asset, GDPR/data minimization, legislation/self-regulation

## 챕터 개요 (3줄 요약)
- 모든 시스템은 목적과 의도치 않은 결과를 가지며, 엔지니어는 그 결과를 숙고하고 해를 끼치지 않을 책임이 있다 — 데이터는 추상이 아니라 사람에 관한 것.
- predictive analytics는 과거의 bias를 codify·amplify하고 자기강화 feedback loop를 만들며, 데이터 수집 자체가 surveillance·privacy 침해 문제를 낳는다.
- 기술은 그 자체로 선악이 아니라 사용 방식이 중요하며, data minimization·self-regulation·인간 존중의 문화 전환이 필요하다.

---

## 1. Predictive Analytics
> 데이터로 사람에 대한 자동 결정을 내릴 때 윤리적 딜레마가 발생한다.

- 날씨·질병 예측과 달리 재범·대출·보험 예측은 개인의 삶에 직접 영향. 'no' 결정 누적은 사회 참여 배제(`algorithmic prison`) — 무죄 추정 원칙에 반함.
- `bias/discrimination`: 학습된 패턴은 opaque. 입력에 bias가 있으면 출력에서 학습·증폭('machine learning is like money laundering for bias'). protected trait(인종·나이·성별 등)과 상관된 proxy(우편번호·IP) 문제. 과거가 차별적이면 codify·amplify — 더 나은 미래엔 moral imagination(인간만 가능) 필요.
- `responsibility/accountability`: 알고리즘 오류 시 책임 소재·항소 가능성. credit score(과거 행동)와 달리 predictive는 '당신과 비슷한 사람'(stereotyping). 통계적이라 개별 케이스는 틀릴 수 있고, 잘못된 데이터의 구제는 거의 불가.
- `feedback loop`: credit score로 채용 → 실직 → score 악화 → 더 실직(하강 나선). 알고리즘 가격 담합 사례. `systems thinking`(사람+컴퓨터 전체)으로 의도치 않은 결과 예측.

---

## 2. Privacy and Tracking
> 사용자가 명시 입력한 데이터 처리는 서비스지만, 부수적 추적은 surveillance가 된다.

- 행동 추적이 기능 개선(검색 랭킹·추천·A/B)엔 유익하나, 광고 기반이면 광고주가 진짜 고객 — 추적이 상세해지고 profile 구축. 이 관계는 `surveillance`로 묘사 가능('data'를 'surveillance'로 치환해보면).
- 디지털화로 위치·관계·구매·건강 데이터 수집이 쉬워짐 — 과거 전체주의 정권도 못 한 감시를 자발적으로 수용(기업이 서비스 제공 명목). 운전·건강 데이터가 보험료에 영향, smartwatch로 타이핑(비밀번호) 추론.
- `consent/freedom of choice`: 추적의 필요성 의문(광고 때문?), 사용자는 데이터 처리를 이해 못 함(privacy policy는 obscure), 일방적·비대칭 관계. GDPR은 'freely given·specific·informed·unambiguous' 동의 요구. 필수 서비스는 opt-out 비현실적(network effect·사회적 비용).
- `privacy`: 모든 걸 비밀로 함이 아니라 '무엇을 누구에게 드러낼지 선택할 권리'(decision right). 감시는 이 권리를 개인→기업으로 이전. privacy setting도 서비스 자체의 무제한 접근은 못 막음.

### Data as Assets and Power
- 'data exhaust'(폐기물 재활용)가 아니라 사용자 활동은 labor로 볼 수 있음. data broker가 비밀리에 거래. startup은 'eyeballs'(감시 능력)로 평가.
- 기업·정부 모두 데이터를 원함(비밀 거래·강제·절도·파산 시 매각·유출). 'toxic asset'/'hazardous material'/'new uranium' — 잘못된 손에 들어갈 위험을 benefit과 균형해야. 미래의 모든 정부를 고려('poor civic hygiene to install technologies that could facilitate a police state'). 지식=권력(감시는 통제력).

### Industrial Revolution 비유
- 정보화 시대는 산업혁명과 유사(성장+생활수준 향상, 그러나 오염·아동노동 등 어두운 면). 규제(환경·안전·아동노동·식품)가 비용을 늘렸으나 사회 전체가 이익. 'Data is the pollution problem of the information age, protecting privacy is the environmental challenge'(Schneier).

### Legislation & Self-Regulation
- GDPR의 `data minimization`(특정·명시 목적, 필요 최소)은 big data 철학(최대 수집·결합·탐색)과 정면 충돌. 약하게 집행됨. 의료 데이터처럼 규제 vs 혁신 기회의 균형 어려움.
- 필요한 것은 문화 전환: 사용자를 metric이 아닌 존엄·agency를 가진 인간으로. self-regulate(신뢰 유지), 교육, privacy(데이터 통제권) 보호. 데이터를 영원히 보관하지 말고 불필요 즉시 purge·최소 수집('Data you don't have is data that can't be leaked'). '사회적 영향을 고려 안 하면 일을 제대로 안 하는 것.'

---

## Summary (핵심 정리 — 책 전체 마무리)
- 데이터는 선을 행할 수도, 큰 해(삶에 영향·항소 어려운 결정, 차별·착취, 감시 정상화, 친밀 정보 노출, 유출)를 끼칠 수도 있다.
- predictive analytics는 과거 bias를 증폭하고 feedback loop를 만든다 — 책임·투명성·구제 메커니즘이 필요하며 데이터는 도구이지 주인이 아니다.
- 데이터 수집 자체가 surveillance·privacy 침해 — 진정한 consent는 드물고 관계는 비대칭. privacy는 '무엇을 드러낼지 선택할 권리'다.
- 산업혁명처럼 정보화의 어두운 면(데이터 = 정보화 시대의 오염)을 직면·해결해야 하며, data minimization과 self-regulation이 출발점.
- 엔지니어는 사람을 humanity·respect로 대하는 세상을 향해 일할 책임이 있다 — 이것이 책 전체를 관통하는 결론.

### 책 전체 챕터 요약
- Ch1 trade-off(OLTP/OLAP, cloud/self-host, distributed/single-node) / Ch2 nonfunctional requirement(performance·reliability·scalability·maintainability) / Ch3 data model(relational/document/graph/event sourcing/DataFrame) / Ch4 storage engine(LSM-tree/B-tree/column/full-text/vector) / Ch5 encoding·dataflow(Protocol Buffers/Avro, REST/RPC/message).
- Ch6 replication(single/multi-leader/leaderless) / Ch7 sharding / Ch8 transaction(isolation·atomic commit) / Ch9 distributed systems의 문제(network/clock/pause) / Ch10 consensus·linearizability / Ch11 batch processing(MapReduce/dataflow/shuffle) / Ch12 stream processing(message broker/CDC/window/join) / Ch13 streaming systems 철학(unbundling/dataflow/correctness) / Ch14 ethics.