# 04. Getting Started with a Data Quality Initiative

## 챕터 개요 (3줄 요약)

- Business case 승인 직후 몇 주는 가장 stressful하며, supplier·tool 선정, discovery 계획, hiring, communication이 병렬로 진행된다.
- 활동을 5개 workstream(supplier/tool selection, planning/management, early remediation, data discovery, tool implementation)으로 조직한다.
- 팀 구성(data quality lead·architect·developer·tester·business user)이 성패를 좌우하며, 영구직 vs contingent 역할을 구분한다.

---

## 1. First Few Weeks After Approval

> 승인은 첫 rule delivery의 출발 신호. 활동들은 sequence가 아닌 동시다발적으로 진행된다(discovery 중 긴급 이슈 발견, design 중 개발 시작, build 중 testing 시작).

### Supplier Selection
- 3rd-party 컨설팅이 가장 흔함(완결된 팀·skill, 다른 프로젝트 팀과 연계, fixed-price 가능). 대안: contract market, 내부 자원/secondment.
- 선정 요소:
  - **Depth of DQ resources**: 대형 household name이라도 DQ depth 부족할 수 있음(현지 자원·경험 확인). 사례: 작은 niche 컨설팅(150명)이 대형사보다 DQ 전문가 多, day rate 높지만 lean·정확해 총비용 저렴.
  - **Cost**: like-for-like 비교(예: defect 처리 한도). 비슷한 가격이면 senior rate가 유리(한 번에 정확).
  - **Ideas/accelerators**(예: SAP inactive supplier 탐지 방법론 → 5-10일 절감), **industry knowledge**(pharma·banking 규제), **track record**(유사 기술·성숙도의 reference).

### Tool Selection (supplier보다 먼저)
- shortlist(≤3): IT benchmark(Gartner·Forrester) + architecture team(기존 SAP/Microsoft 투자 고려).
- 선정 요소(Table 4.1): **capabilities(최우선)**, cost, 다른 data management tool과의 fit(suite 통합), 시스템 architecture fit, 공급사 sustainability, licensing model(capex vs opex/SaaS), 사용 방식 fit(user당 vs capacity).
- 참여자: DQ lead, governance lead, data owner/steward, architecture, compliance. procurement과 scoring 메커니즘.

### 기타 초기 활동
- **Data discovery 계획**: senior stakeholder 미팅 조율(시간 소요), profiling 위해 tool 셋업·DB 연결. IT architectural review board 승인, infra 통합(SaaS면 firewall, 비SaaS면 VM). 4-6주 소요.
- **Hiring**: 영구직은 가능한 빨리 시작, secondment·internal transfer로 신속 충원.
- **Communication**: 최초 소통은 최고위(CDO)가 — data owner/steward, IT leader, functional leader, process/system owner에게 scope·기여 요청.

---

## 2. Data Quality Workstreams

> Workstream으로 나누면 명확한 scope을 사람에게 할당하고 주간 단위 진척·동기부여 가능.

```text
Supplier/tool selection   supplier·tool 선정 후 종료
Planning/management       전 기간 지속, phase별 focus 변경
Early remediation         초기 긴급 이슈 → 이후 main remediation(Ch8)으로 전환
Data discovery            stakeholder 미팅·rule 식별 → 이후 design·testing으로 전환
DQ tool implementation    tool 셋업·infra → 이후 build로 전환
```

### Early Remediation Workstream (quick-wins)
- discovery 중 긴급 이슈 발견(예: HR bonus 계산 데이터가 4주 내 board 보고 필요). 무시 말되 **별도 workstream으로 scope 엄격 통제**.
- 핵심팀 역할은 **coordination·communication·SME로 제한** — 영향받는 함수의 인력을 식별, 접근법 합의(예: line manager 이메일로 누락 데이터 수집), 조율.
- 사례 교훈: HR headcount 데이터(잘못된 org unit, worker 분류 오류)를 DQ팀이 직접 반복 추출 → rule 구현 지연. HR팀에 추출법을 가르쳤어야 함. 관계 강화엔 도움.
- 필요 skill: project manager + data quality analyst 결합. business case에 contingency 확보 권장.

### Workstream 간 상호작용 (Table 4.4)
- early remediation → discovery: 데이터 深 이해로 rule 식별·정제(주간 공유).
- discovery → tool implementation: 핵심 source system 조기 식별.
- discovery → planning: scope 급변 대응(예: supply chain → commercial로 초점 이동).

---

## 3. Identifying the Right People

> 성패는 팀의 skill·knowledge·motivation·협업에 달림. 구현 phase엔 팀 확대, BAU 전환 시 contingent 인력 이탈 → knowledge transfer 필수.

### Business case role ↔ governance role 매핑 (Table 4.5)
```text
Project manager      ↔ Data quality lead (PM 부재 시 lead가 겸임)
Business user        ↔ owner/steward/champion/producer/consumer (spoke)
DQ architect         = 이니셔티브 한정, contingent
DQ developer         = contingent (일부는 BAU managed service로 잔류)
DQ tester            = contingent
```

### 역할별 핵심 attribute
- **DQ lead** (가장 중요): 직접 보고 안 하는 팀 leading·influence, DQ/governance SME(고성과자 기준 2년 충분), business acumen, project management, supplier 관계 관리.
- **DQ architect**: 선정된 tool 경험(개발자 책임·estimate 검증), leadership·communication, data architecture(tool을 복잡한 system에 통합), business acumen(rule을 기술 언어로 번역).
- **DQ developer**: tool 경험(ETL·rule·report 구현), business acumen(예: 매출 \$100M 조직에 supplier 100만 개면 이상 감지), data visualization skill(ETL/rule용과 report용 프로필 분리 가능).
- **DQ tester**: business acumen(rule scope별 record 수 검증), 극도의 attention to detail(20개 중 1개 차이도 끝까지 추적), written/verbal communication(개발자에 이슈 설명·business user training).
- **Business user**: Ch2의 spoke 역할들. discovery·early remediation에 일상 업무와 병행 참여, partner이자 customer.

---

## Summary (핵심 정리)

- Business case 승인 후 첫 몇 주는 대체로 기대만큼 생산적이지 못함 → 철저한 준비가 중요.
- 3rd-party partner(자원·tool) 선택과 올바른 팀 hiring이 성패를 정의한다.
- 이니셔티브를 manageable workstream으로 분해해 대부분이 적절한 속도로 진행되게 한다.
- 초기 주가 성공적이면 data discovery phase가 잘 set up됨 → 다음 챕터(5장)가 그 주제.
