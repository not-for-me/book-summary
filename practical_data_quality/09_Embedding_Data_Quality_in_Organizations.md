# 09. Embedding Data Quality in Organizations

## 챕터 개요 (3줄 요약)

- 일회성 remediation의 benefit을 지속시키려면 데이터 수집 방식을 바꾸고, Ch3~8 활동을 축소된 BAU(business-as-usual) 형태로 이어가야 한다.
- 이슈 재발 방지(retraining·validation·interface·MDM), rule 변경 추적, day-to-day remediation 전환을 다룬다.
- 단일 이니셔티브를 18개월+ roadmap으로 확장해 조직 전체의 data quality를 변혁한다.

---

## 1. Preventing Issue Re-occurrence

> 원인 이해 없는 일회성 cleanse는 재발 → "stable door": 검출(rule)·말 찾기(remediation)·문 닫기(재발 방지).

### 변경 유형 (Table 9.1)
- **Retraining**: validation 불가·판단 필요한 경우.
- **System validation**: "hard and fast" rule에 최선. 최신 DQ tool은 API로 실시간 validation(DQ tool이 master).
- **Interface/MDM**: 다중 시스템이 공통 데이터 사용 시.
- **3rd-party 데이터 품질 개선**, **business partner self-service 입력 개선**(supplier onboarding pack, customer tooltip, employee 교육).

### Human Error & Short-Horizon Reporting
- training·validation에도 human error 잔존(예: color 필드에 "Yes" 입력, validation은 email 존재까지 못 확인).
- **Short-horizon reporting**: date filter로 최근 2일 생성/변경 데이터만 → 현재 operative 성과 파악, user ID로 특정 팀 targeted training.

---

## 2. Ongoing Rule Improvement

> rule은 고정 불가 — 세법 등은 10년 가지만, business 밀착 rule은 수주 내 변경(예: phone network 2G/3G → 4G/5G).

### Rule 변경 식별 전략 (Table 9.2)
- ARB(architecture review board) 참여(DQ 영향 표준 질문), system governance committee, IT security team, IT ticketing(go-live 단계라 advance notice 짧음), data steward network, town hall, 조직 template·business case에 data quality 삽입.
- 변경 delivery: change owner가 자금 / **영구 capacity 보유**(전형 수요의 ~80% 권장, 대형 프로젝트만 추가 budget).

### Rule 업데이트
- live rule을 프로젝트가 go-live 전까지 건드리지 않도록 change management(dev→test→live). data steward가 design doc·test 증거 리뷰.
- score "jump" 설명 필요(예: raw material 50개 제외로 77%→90% 하루만에 변경 시 ticket 방지).

---

## 3. Transitioning to Day-to-Day Remediation

> 집중 프로젝트 종료 후에도 남은 일부 교정을 BAU 팀이 이어받아야 함.

### 성공 요건
- **Sufficient time**: 팀이 DQ 작업용으로 sizing 안 됨. 일부는 자동 해소(예: remittance email 개선 → query 감소로 capacity 생김), 아니면 우선순위 협의·증원·escalation.
- **Knowledge of good/bad data**: rule·실패 원인 명확 이해 + 깊이(process·analytics·compliance 영향). 조직 purpose와 연결해 동기부여(예: cold storage 의약품 배송시간).
- **Cooperation from partners**: supplier/customer/employee 입력 의존 → MDM tool(Informatica MDM, SAP MDG)의 guided form·validation·자동 process. 미투자 시에도 Excel form에 최대 validation.
- **Sufficient access**: 단건이 아닌 governed mass change 권한. 통제는 log 리뷰·supervisor 승인으로 mitigate.
- **Right culture**: 데이터에 자부심·ownership, 실수는 학습 기회, 조직 위한 타협. 진척 celebration, 데이터 sub-team 배정, 실수 공개 지원.

### 성공적 transition 계획
- **Documenting remaining issues** (Table 9.3 예: hospital hierarchy ID): 상태(시작/교정/잔여, 우선순위), 선택한 approach·작동 여부, **next steps**(예상 pace 포함).
- **Owning BAU team 식별**: 거부 시 governance forum escalation, 미해결 이슈는 accepted risk로 문서화.
- **Knowledge transfer**: 전원 참여(record-by-record 교정), 미참여 팀은 도구·배경까지. elevated access·DQ tool 접근.
- **BAU status reporting**: governance forum·부서 리더 미팅·board pack에 가시성 유지(visibility = 성공).

### 성공 지표 (~4주 후)
- remediation pace(대부분 on track 여부), 신규 이슈 발생률, 관련 business KPI 추이. 실패 원인: 자원·access·정치(리더 미지원)·engagement.

---

## 4. Continuing the Data Quality Journey

> cyclical process — BAU 전환 후 새 이니셔티브 scope으로 복귀.

### Roadmap
- 첫 이니셔티브 완료 전 두 번째 scope 시작 → 자원·지식 seamless 이전, onboarding 시간 최소화, 유휴 시간 활용. **rule demo 가능 시점**에 다음 scope 시작.
- 18개월+ roadmap → budget cycle에 포함(net new가 아닌 allocated funds).

### 다음 이니셔티브 선정
- 첫 discovery에서 우선순위였으나 제외된 영역 재방문(stakeholder와 월간 cadence로 "warm" 유지).
- **Complementary**(첫 이니셔티브와 overlap): stakeholder 지식, 재사용 데이터·통합, rule 강화(예: payment term = contract 일치), 동일 시스템 access.

### 지원 확보 (2번째가 더 쉬움)
- 기존 report demo(계단에서 2분 데모로 임원 지원 획득 사례), business stakeholder 공동 발표, 정당성 갖춘 benefit.

### 추가 이니셔티브 미승인 시
- 이사회 변경·trading headwind·규제 변화로 조기 종료 가능(DQ는 benefit 입증 어려워 타깃). 첫 이니셔티브 legacy 보호: BAU 모니터링 지속, support 팀의 도구 이해, owner/steward의 "keep the lights on".

---

## Summary (핵심 정리)

- 예산 이니셔티브 후 DQ 지속 방법: 재발 원인 최소화, business change 추적·rule baseline 관리, remediation을 BAU에 embed, 단일 이니셔티브를 장기 roadmap으로 전환.
- Ch2의 data quality improvement cycle을 전부 거쳤다.
- 다음 챕터(10장)는 핵심 best practice·흔한 실수, 그리고 향후 innovation을 다룬다.
