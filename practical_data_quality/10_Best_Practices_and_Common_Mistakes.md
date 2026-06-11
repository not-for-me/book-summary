# 10. Best Practices and Common Mistakes

## 챕터 개요 (3줄 요약)

- 여러 조직 경험에서 도출한 핵심 best practice 4가지(신규)와 책 전반의 7가지, 그리고 흔한 실수들을 정리한다.
- 실수 대부분은 best practice의 역(특히 benefit tracking·rule testing 미흡)이며, 비실용성·IT 주도 rule·일회성 remediation·결과 접근 제한·silo가 주요 실패 원인이다.
- LLM이 향후 10년간 data quality 작업을 변혁할 것으로 전망한다.

---

## 1. Best Practices (신규 4가지)

### Manage DQ primarily at the source
> ultimate source system(데이터가 생성된 곳)에서 평가·교정해야 함. secondary source(data warehouse 등)는 transform으로 이슈를 숨기거나 악화.
- secondary 교정 위험: 다른 secondary로 source 데이터 계속 사용, 불완전·오해 소지 fix(특정 시점만·aggregate level만).
- 단, 마감 압박 시 임시 workaround 허용 가능(부정확성 명시 + source 이슈 보고·추적). secondary는 completeness·format·transformation·consistency 모니터링은 필요.

### Implement supporting governance meetings (Table 10.1)
```text
DQ Steering Group (분기)      CDO 의장, 전 이니셔티브 감독·핵심 결정·escalation
DQ Working Group (2주)        Governance lead 의장, 조율·remediation 우선순위·escalation
DQ Initiative Steering (월)   DQ lead 의장, 단일 이니셔티브 on time/budget
DQ Initiative Project (주)    PM 의장, day-to-day 실행
```
- 역할: risk/issue 평가·escalation, 일관된 대응, 중복 접촉 방지(예: supplier 한 번만 연락, 3rd-party 요청 통합).

### Organization-wide education program
- **Generic training**(전원, ≤15분): poor DQ 영향·정의, 모든 role의 책임, 이슈 제기·도움 요청. ("데이터는 data role만의 것"이라는 주장에 강력 반대.)
- **Role-specific training**(spoke roles): hub-and-spoke 모델, 각 역할 심화. testing phase에서 steward engagement↑ 효과.

### Leverage data steward & producer relationship
- spoke의 make-or-break 두 역할. steward는 producer를 advocate(training·시간 확보), remediation 일정 추정, cross-functional 협력 중재.

### 책 전반 best practice (Table 10.4)
- benefit 추적(Ch8), business strategy로 시작(Ch5), rule scope 정밀 정의(Ch6), rule 철저 testing(Ch6), stakeholder별 report(Ch7), remediation 전 재우선순위화(Ch8), 재발 방지(Ch9).

---

## 2. Common Mistakes

### Failure to implement best practices
- **benefit tracking 실패**(가장 불편한 시점에 필요하나 필수 — 추후엔 데이터 확보 불가). **rule testing 부족**(가장 흔한 실수 — false failure는 detractor에게 "탄약" 제공, 도구 무력화).

### Lack of practicality
- 경직된 접근 회피. 유연성 trigger: 긴급 이슈 조기 remediation, 조직/전략/정치 변화(예: 인수로 직원 2배 → re-plan, IT 전략 변경 → tool 재검토).

### Technically driven rules
- business SME와 격리된 IT 주도 → 피상적·기술 중심·불완전·잘못된 우선순위 rule. 예: 필드 길이 체크(integration tool에 내장 가능), scope 없는 mandatory 체크.

### One-off remediation
- 집중 후 이동하는 문화는 DQ에 부적합 → 일시적 개선만. 장기 일관 접근 필요.

### Restricting access to results
- 결과 제한은 competition·peer pressure를 없애 remediation 지연. open 유지 권장.

### Avoid silos
- 데이터는 연결됨(customer↔order↔product↔supplier↔PO). 예: customer experience 이슈의 root cause가 supply chain 데이터인데 budget 없음 → commercial 팀이 도와야.

---

## 3. The Future of Data Quality Work (LLM)

> ChatGPT·GPT-4 등 LLM이 향후 data quality를 변혁. Microsoft Copilot 등 제품 통합 진행.

### LLM use cases
- **rule code 자동 생성**: developer 의존 감소, business user가 "LLM-friendly" 설명 작성. 강한 business glossary(term↔metadata 매핑)가 있으면 단순함 유지. (오늘날 ChatGPT로 C# 코드 생성 가능하나 가정(월=30일 등) 검증 필요.)
- **rule 설명 자동 생성**(code→business description), **ETL·duplication·inactive 로직 가속**.
- **profile+definition 기반 rule 추천**(strategy·definition·profile 종합 시 business value 있는 제안).
- **remediation 가속**: 내부 inconsistency 교정 제안, OCR 문서 스캔, 외부 authoritative resource 대조, partner와 chat로 누락 데이터 수집.

### 시사점
- 단계는 동일하나 대부분 가속 → 비용↓·착수 용이. data quality lead는 LLM 활용·partnering 경험 필요. 새 유형의 developer(LLM 초안+debug).
- **poor data quality는 LLM benefit을 무력화**(definition 누락 시 rule 무용, customer master 불량 시 AI 고객 서비스 실패). → DQ 수요 더 빠르게 성장.

---

## Summary (핵심 정리)

- 따르면 최선의 결과를 주는 best practice와, 피하지 않으면 목표 달성을 막는 흔한 실수를 정리했다.
- LLM의 영향은 향후 수년간 transformative할 것으로 예상된다.
- 저자는 16년간 잘 지원된 이니셔티브가 조직의 전략 목표 달성 능력을 진정으로 변혁하는 것을 목격했으며, 이 책이 독자의 첫 이니셔티브 성공을 돕길 바란다.
