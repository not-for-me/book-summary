# 08. Data Quality Remediation

## 챕터 개요 (3줄 요약)

- Remediation은 데이터를 fit-for-purpose 수준까지 교정하는 가장 도전적이지만 benefit이 실현되는 단계로, tranche 단위 cyclical process다.
- 4단계(prioritize → identify approach → understand effort/cost → remediate)로 진행하며, collaborative voting으로 우선순위를 정한다.
- 7가지 remediation approach(effort 순)를 이슈별로 선택·조합하고, governance로 추적하며 benefit을 측정한다.

---

## 1. Overall Process (cyclical)

> 모든 이슈를 동시에 처리 불가 → tranche로. 한 cycle 완료 후 다음 우선 tranche 반복.

```text
Prioritize          → 중요 실패 식별
Identify approach   → 교정 방식(수동/3rd-party 수집/rule 기반 등)이 effort 결정
Understand effort   → approach 기반 effort·cost 추정
Remediate          → 자원 배분(high-priority·low-effort 우선), 교정·모니터
```

---

## 2. Prioritizing

> 첫 Rule Results Report는 압도적(대기업은 rule당 25만+ 실패 흔함, 예: 18M 고객 reward 카드의 1% = 막대한 양). 자원은 한정 → 우선순위 필수.

### Benefits 재검토
- business case의 benefit이 여전히 top priority인지 확인. 실제 실패 record 수가 계산을 바꿈(예: 예상 6만 → 실제 1만이면 benefit 15%로 하락).
- 진행 중 식별된 신규 rule이 가장 가치 있을 수 있음(통찰력 있는 stakeholder 그룹 발).

### Collaborative voting (선호)
> 전수 \$ 정량화는 비선호(시간 소모, 정성 요소 무시). 대신 stakeholder 협업 합의.

```text
Identify stakeholders  영역별 영향 파악 + 우선순위 결정권자
Vote for priorities    이슈 제시 후 top 5 순위 투표 (rank 1~5 → score 5~1)
Formalize priorities   전체 미팅에서 합의 (가장 어려움)
```
- 초기 미팅: 맥락 brief, 점수·실패수 제시, 도구 training, 영향 질의(예: SAP delivering plant 오류 → Canary Islands 제품을 본토에서 발송).
- 투표 집계(Fig 8.3/8.4): 동점 시 rank 분포 확인. **consensus 있는 rule 선호**(많은 사람이 포함). 단, 핵심 stakeholder engagement 위해 조정 가능.
- 합의 실패 시: escalation(steering committee) 또는 tactical(축소 목록으로 시작).

---

## 3. Remediation Approaches (Table 8.4, effort 순)

> 이슈마다 1개 이상 approach 선택, 종종 조합. effort 낮은 순으로 사고하되 cost도 고려.

```text
1. Applying rules to data        Low   다른 데이터로 값 derive (예: Canary 고객 → Canary plant)
2. Collect from internal         Low   institutional knowledge (예: cost center owner)
3. Copy from another system      Low   동일 데이터가 타 시스템 존재 (interface/MDM로 영구화)
4. Match & merge with 3rd party  Medium D&B 등 unique ID로 매칭 (setup 高, record 무관)
5. From internal documentation   Medium 종이/PDF, OCR로 자동화 가능
6. Collect manually              High  supplier/customer 직접 연락 (최후 수단)
7. Online search                 High  웹/API (소량·타 활동과 결합 시 유용)
```

- 이슈별 approach 매칭(Table 8.5 예): 결제조건 불일치→approach 5, bank details→5+6, supplier hierarchy→4(D&B 90% 매칭, spend 1.4%↓), DOB 오류→6, org unit→1, product weight→3(MES interface).
- **SME와 협의 필수**(강요 시 저항), 단 운영자는 모든 가능성을 모를 수 있음(타 시스템·D&B 서비스 등).

### Moving to BAU
- 자동/대량 교정이 100% 못 함 → 어려운 20%는 cost가 benefit 초과할 수 있음. 80% value approach 적용 후 나머지는 BAU.
- 사례: supplier bank details 65% 누락 → approach 5로 80% 해결, 나머지는 ERP에서 purchasing block(거래 시 organically 수집), ~100개만 직접 연락.

---

## 4. Understanding Effort & Cost

> approach별 effort·cost 추정. 매우 어려우면 더 쉬운 이슈로 re-prioritize. **momentum이 중요** — 과도한 분석은 momentum 상실.

```text
Approach          People effort    Non-people cost   Level
1 rules           Low              Low(ADM)          Low
2 internal collect Moderate         None              Moderate
3 copy system     Low              Low(ADM)          Low
4 match&merge     Low              Medium(record당)   Moderate
5 documentation   Moderate(OCR시Low) Low(OCR시 tech↑)  Moderate
6 manual          Highest          None              High
7 online          High             None              High
```
- 내부 인건비는 간단 추정(일수·timeline)으로 충분. 외부 비용(3rd-party)은 대안 대비 정당화 필요.

---

## 5. Governing Remediation (Table 8.7)

> 프로젝트처럼 governance 필요 — 추적·보고·risk/blocker 관리·BAU 전환. 초기엔 formal(처음이라 best practice 부재), 성숙 시 축소.

- **Plan**: 자원·timescale·dependency(예: 5개 이슈 모두 해결 후 1회 upload, 월말 close 시 자원↓).
- **Governance meeting**: 진척 review(최신 reporting), plan update, risk/issue, approach 변경.
- **Regular reporting**: summary·성과·milestone·risk.
- **Prevention of re-occurrence**: 영구 해결책, root cause, validation 추가/재교육/주기적 remediation (Ch9 상세).

---

## 6. Tracking Benefits

> remediation에만 몰두해 stakeholder 관리를 놓치기 쉬움. 약속한 benefit 실현 입증이 중요 → 다음 영역 확장·투자 유도.

- 첫 단계: 현재 vs 원래 data quality 대비(예: 평균 65%→85% 6개월). 따라서 historic view/trend 또는 사전 snapshot 필요.
- 항상 추적 필요한 건 아님(이미 가시적 모니터링되는 경우, 예: supplier 결제 backlog).
- **정량 예**: remittance email 누락 → query율 5%→1%로 비용 \$120k→\$25k(\$95k 개선). business case 외 benefit(예: HR팀 2 FTE 재배치)도 강조 — budget holder와 긴밀 관계 필요. approach 2(extrapolate)·3(benchmark) 사후 검증으로 신뢰도↑.
- **정성**: 직원 survey 재실시, business case risk 완화 증거(NPS, 고객 comment 변화).

---

## Summary (핵심 정리)

- remediation은 가장 도전적 — 한정 자원을 조직이 믿는 핵심 우선순위에, 가장 효과적 approach로 투입하는 법을 다뤘다.
- 진척을 stakeholder에 보여주고 business case의 benefit과 연결하면 다른 영역으로 확장할 mandate를 얻는다.
- benefit 지속을 위해선 데이터 관리 방식을 영구 변경하고 process에 embed해야 함 → 다음 챕터(9장)가 그 문화 전환의 best practice.
