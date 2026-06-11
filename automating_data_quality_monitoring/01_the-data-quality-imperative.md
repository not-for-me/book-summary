# 01. The Data Quality Imperative

## 챕터 개요 (3줄 요약)
- data quality 문제는 Equifax credit score 오류, COVID test 손실 등 막대한 비즈니스 손실을 일으키며, 대부분의 issue는 silent하게 가치를 파괴한다.
- cloud data warehouse, analytics democratization, ML/AI, modern data stack 트렌드로 high-quality data 의존도는 높아지지만 동시에 complexity로 품질 확보는 더 어려워졌다.
- 일회성 fix가 아닌 continuous data quality monitoring이 필요하며, 본서는 unsupervised ML 기반 자동화를 핵심 해법으로 제시한다.

---

## 1. High-Quality Data Is the New Gold
> data는 "new oil/gold"라 불리지만, 오직 high quality일 때만 그렇다 — bad data is worse than no data.

- **Data-driven 기업이 disrupter**: Amazon(recommendation, real-time pricing), Capital One(cloud migration, underwriting)처럼 data+software 교차점이 경쟁 frontier. 단, 신뢰 불가능한 data 위에서는 무의미.
- **Analytics democratization**: 누구나 no-code로 dashboard·report를 self-serve. data가 중앙팀에서 business line 가까이로 분산됨. 단 trust 없으면 data engineering팀에 fire drill backlog만 쌓임.
- **AI/ML differentiator**: model은 training distribution과 production data가 일치할 때만 잘 작동하고, out-of-distribution data엔 처참히 실패. data quality가 model 성패를 가른다.
- **Generative AI**: unstructured raw data를 직접 ingest하지만 data quality는 여전히 필수 — structured data를 prompt에 결합하고, 자동화 추적을 위한 high-quality log가 필요. unstructured data의 issue 탐지엔 오히려 automated ML이 더 중요.
- **Modern data stack**: SaaS vendor 조합으로 과거 100명 data engineer 작업을 대체. 단 data quality tool이 빠지면 투자 가치 훼손, 특히 migration 직후 bad state 위험.

---

## 2. More Data, More Problems
> high-quality data는 그 어느 때보다 달성하기 어렵다 — 더 많은 complexity, 더 빠른 속도, 더 적은 guardrail.

- **Data factory 비유**: warehouse가 아니라 raw material(streaming dataset, DB replica, API extract)을 product로 변환하는 factory. ETL(Matillion, Fivetran), orchestration(Airflow), transformation(dbt, Spark, SQL)이 machine 역할.
- **Factory floor에서 발생하는 issue**: broken machines(도구 고장), scheduling errors(순서·cadence 오류), poor raw materials(upstream 품질), incorrect parts(SQL/Spark 코드 오류), incorrect settings(config 실수), botched upgrades, communication failures.
- **Data migration**: on-prem→cloud, cloud→cloud, DB 버전 변경 모두 issue에 취약. 예: mainframe의 birthdate integer offset(1900 기준)을 cloud가 Unix(1970) 기준으로 해석해 모든 생일이 미래로 밀림 → marketing email 미발송.
- **Third-party data**: weather, maps, CPG catalog, fraud scoring 등. provider의 mistake뿐 아니라 API·format 변경이 흔한 문제 원인. 명확한 data contract 없으면 사전 통지 보장 안 됨.
- **Company growth and change**: 거의 모든 data는 처음엔 high quality지만 entropy처럼 시간이 갈수록 degrade — new features, bug fixes, refactors, optimizations, new teams, outages가 원인.
- **Exogenous factors**: user behavior, global event, competitor action 등 통제 불가 요인. data quality issue처럼 보이지만 실제 외부 변화일 수 있음 (예: COVID-19로 Chicago taxi 거리 급감, Zillow 모델 실패).

---

## 3. Why We Need Data Quality Monitoring
> software와 달리 data는 chaotic하고 끊임없이 변하므로, 오직 production에서 holistic하게만 test 가능하다.

- software는 controlled QA·unit test로 한 번 통과하면 끝이지만, data는 외부 요인에 의존하므로 noise 속에서 진짜 signal을 걸러내야 함.
- **issue 비용은 시간이 갈수록 급증**: 원인 후보 증가(linear), 변경 context 감소, backfill 비용 증가, 오래된 issue가 downstream에 "normal"로 굳어져 fix가 새 incident 유발.
- **Data scars**: incident 이후 repair되지 않으면 특정 기간 record가 anomalous한 scar로 남음. ML model이 scar 기간 feature를 underweight하고, fix 시 data leakage 위험. analytics에도 복잡한 exception handling·data amnesia 초래.
- **Data shocks**: issue 발생 시 model이 untrained distribution에 "shock"받아 부정확 예측. fix 시점에도 또 한 번 shock 발생(초기 shock만큼 클 수 있음). 오래 방치할수록 scar는 깊고 shock은 커짐.
- 결론: data quality는 one-off project가 아니라 continuous monitoring initiative여야 함. trust는 backfill하기가 data보다도 더 어렵다.

---

## 4. Automating Data Quality Monitoring: The New Frontier
> 수천 table·수십억 record 규모에선 manual inspection도, legacy test 작성도 전체 warehouse엔 비현실적이다.

- 중요 table엔 test·metric tracking이 유효하지만 전체 data warehouse엔 확장 불가.
- **해법: unsupervised ML 기반 자동화** — manual setup 거의 불필요, warehouse 전반에 쉽게 scale, data 변화가 quality issue인지 threshold를 자동 학습, test로 예상 못한 unknown unknowns까지 탐지.
- **ML의 과제**: model 구축 자체가 복잡하고, over/under-alerting 없이 real-world data에 작동시켜야 함. triage용 notification, data toolkit integration, 장기 deploy·관리 plan 필요.
- 본서는 이 모든 과제에 대한 advice·tool을 제공하며, ML 기반 data quality monitoring을 modern data stack의 핵심 breakthrough로 본다.

---

## Summary (핵심 정리)
- data quality issue는 대부분 silent하게 가치를 파괴하며, data 의존도가 높아질수록 그 비용도 커진다.
- modern data stack·migration·third-party·company change·exogenous factor가 data factory 전반에서 끊임없이 품질을 저하시킨다.
- data scar(누적된 손상)와 data shock(발생·복구 시점의 충격)이 시간이 지날수록 trust를 erode시킨다.
- 따라서 data quality는 one-off fix가 아닌 continuous monitoring이 필요하며, 본서의 핵심 해법은 unsupervised ML 기반 자동 모니터링이다.
