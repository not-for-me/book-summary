# 06. Fixing Data Quality Issues at Scale

## 챕터 개요 (3줄 요약)

- 소프트웨어 인시던트 관리(DevOps/SRE)의 모범 사례를 데이터에 적용한 "데이터 신뢰성 라이프사이클"로 데이터 다운타임을 대규모로 해결하는 법을 다룬다.
- 인시던트 탐지·대응·근본원인분석(RCA)·해결·무비난 사후검토(postmortem)의 단계별 절차와 5단계 RCA 방법을 설명한다.
- 인시던트 커맨더 역할, 테스트·CI/CD·디스커버리·옵저버빌리티 기반 사전 예방 스택, PagerDuty 사례를 제시한다.

---

## 1. Fixing Quality Issues in Software Development (소프트웨어에서 배우기)

> 데이터 팀은 바퀴를 재발명할 필요 없이 DevOps·SRE의 인시던트 관리에서 영감을 얻을 수 있다.

- DevOps 라이프사이클 8단계: Plan → Code → Build → Test → Release → Deploy → Operate → Monitor (반복).
- 인시던트 관리: 일상 엔지니어링 워크플로우에서 발생하는 이슈를 식별·근본원인분석·해결·분석·예방하는 과정.
- 반응적(reactive) 대응 관행이 분석 도입을 가로막아 왔으며, SW 모범 사례로 선제적·확장 가능한 접근이 필요.

---

## 2. Data Incident Management (데이터 인시던트 관리)

> DevOps 라이프사이클에서 영감받은 "데이터 신뢰성 라이프사이클"로 파이프라인 성능·신뢰성을 관리한다(탐지→대응→RCA→해결→postmortem).

### Incident Detection
- 데이터를 프로덕션 전 테스트하되, bad data는 새어 나가므로 모니터링·알림(Slack/Teams/PagerDuty 등)으로 탐지.
- 이상 탐지(anomaly detection)는 엔드투엔드(웨어하우스·레이크·ETL·BI)로 구현 시 가장 가치 있음.
- 단, 이상 탐지는 "도구이지 만능 해결책(silver bullet)이 아님" → 단일 장애점이 됨. 무엇이/누가/왜/어디서 깨졌는지 추가 분석 필요.

### Response
- 좋은 대응은 효과적 소통에서 시작·끝남(PagerDuty·Slack 자동화). Runbook(서비스 사용·이슈 안내)과 Playbook(인시던트 처리 단계) 작성.
- SRE의 on-call: incident responder + incident commander(작업 할당·정보 종합·상하류 소비자 소통).
- 메타데이터 + 엔드투엔드 리니지로 영향받는 팀을 신속 파악·통지.

### Root Cause Analysis (RCA)
- 대부분의 데이터 문제는 3가지 중 하나: (1)유입 데이터의 예상치 못한 변경, (2)변환 로직(ETL/SQL/Spark) 변경, (3)운영 이슈(런타임 오류·권한·인프라·스케줄).
- Amazon "5 Whys" 프레임워크로 근본 원인 탐색. 단일 원인은 드물다.

```
   5-step RCA:
   1) Look at LINEAGE   -> find most upstream broken node
   2) Look at CODE      -> logic that built the table
   3) Look at DATA      -> which records/segments/time wrong?
   4) Look at OPS ENV   -> ETL logs, errors, schedule
   5) Leverage PEERS    -> ownership/usage metadata
```

- Step 1 Lineage: 필드 수준(field-level) 리니지가 이상적, BI 리포트·ML 모델·reverse-ETL 포함.
- Step 2 Code: 최근 갱신 코드·필드 계산식·로직 변경·ad hoc write/backfill 확인.
- Step 3 Data: 전체/일부 레코드, 특정 기간·세그먼트, 스키마/단위 변경 여부 점검(예: Twitter source의 null rate 급증 분석).
- Step 4 Ops Env: 잡 오류·지연·장기 쿼리·권한/네트워크/스케줄 변경 점검(Airflow 로그).
- Step 5 Peers: 과거 유사 이슈·데이터셋 소유자·사용자에게 문의.

### Resolution
- 초기 해결(initial: 파이프라인 일시정지/circuit break)과 최종 해결(final: 근본 원인 영구 수정)로 구분.
- 진행 상황을 전용 채널(Slack/JIRA/Wiki)에서 지속 공유, 해결 후 postmortem 예약.

### Blameless Postmortem
- "잘못은 사람이 아니라 시스템에 있다" → 결함·인간 허용(fault/human tolerant) 시스템 지향.
- 모범 사례: 모든 것을 학습 경험으로 프레이밍(blameless), 미래 대비 점검(runbook·모니터링 갱신), 문서화·공유, SLA/SLI/SLO 재검토.

---

## 3. Incident Response and Mitigation (대응과 완화)

> 테스트는 "known unknown"의 약 20%만 커버하므로, 나머지 80%를 위한 사전 예방이 필요하다.

- 라이프사이클의 Detect·Resolve는 "반응적", Prevent는 "선제적" 단계.
- 데이터 신뢰성 스택 4요소: Testing + CI/CD + Discovery + Observability → 중앙·분산·하이브리드 어떤 아키텍처에도 적용 가능.

```
   Reactive            Proactive
   [Detect] -> [Resolve] -> [Prevent]
                            = Testing + CI/CD
                              + Discovery + Observability
```

### Establishing a Routine (인시던트 커맨더)
- Incident Commander 책임: 조기·빈번한 플래깅, 영향 자산 기록, 노력 조율·역할 할당, runbook 배포, 심각도·영향 평가.
- 주간/일간 순환 배정 권장. 주로 문화적 과정이며 자동화·교육으로 보완.

### 4단계 트리아지
- Step 1 — Route notifications: 팀 구조(분산형 vs 중앙형)에 맞게 전용 Slack 채널·PagerDuty/Opsgenie로 알림 라우팅.
- Step 2 — Assess severity: 상태 태깅(fixed/expected/investigating 등). "유령 데이터(phantom data)" 주의 — 안 쓰는 데이터에 시간 낭비 방지, 리니지로 중요 자산 식별.
- Step 3 — Communicate status: runbook 따라 책임 분담, 상태 페이지로 실시간 공유. on-call 방식 또는 테이블별 담당 방식.
- Step 4 — Define SLOs/SLIs: SLI 예시 — 인시던트 수(N), TTD(Time To Detection), TTR(Time To Resolution).

---

## 4. Case Study: PagerDuty

> 디지털 운영 관리 플랫폼 PagerDuty의 DataDuty 팀이 자사 제품으로 데이터 인시던트 관리를 수행한 사례.

- 스택: PagerDuty(자사 제품), Snowflake, Fivetran, Segment, Mulesoft, AWS, Databricks + ML 기반 데이터 옵저버빌리티.
- 도전: SaaS·구조화/비구조화·다양한 주기의 데이터를 다루며, 빠르게 변하는 비즈니스 요구에 정확·신속 대응(agile).
- 모범사례 #1: 인시던트 관리가 데이터 라이프사이클 전체를 커버(파이프라인 품질 체크로는 데이터 트렌드 이슈를 못 잡음).
- 모범사례 #2: 노이즈 억제(signal-to-noise 최소화) — 한 인시던트의 여러 알림 묶기.
- 모범사례 #3: 자산·인시던트 그룹화로 지능적 라우팅(Airflow 알림도 PagerDuty 경유, 핵심 자산은 에스컬레이션 정책).

---

## Summary (핵심 정리)

- 대규모 파이프라인 수정을 위해 반복 가능한 인시던트 관리·RCA·신뢰성 워크플로우에 투자해야 한다.
- 4단계: 핵심 파이프라인 인시던트 관리 프로그램 도입, 이상 탐지를 더 큰 탐지 전략의 일부로 활용, 철저한 RCA·영향 분석, 테스트·CI/CD·옵저버빌리티로 선제 대응.
- 다음 장에서는 해결·예방의 핵심 도구인 엔드투엔드 리니지 시스템을 직접 구축하는 법을 다룬다.
