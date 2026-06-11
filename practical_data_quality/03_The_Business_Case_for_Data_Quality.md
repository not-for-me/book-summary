# 03. The Business Case for Data Quality

## 챕터 개요 (3줄 요약)

- Data quality 이니셔티브의 가장 어려운 단계인 business case — 비용 추정은 비교적 쉽지만 benefit 정량화가 본질적으로 어렵다.
- 비용(people + non-people)을 모델링하고, 정량적 benefit 3가지 접근법(payback 충당 / 추출·extrapolate / top-down benchmarking)과 정성적 benefit(risk avoidance)을 제시한다.
- 승인 board의 전형적 challenge("Excel로 충분", 운영비 주인, 과도한 비용, "왜 DQ tool 필요?")에 대한 대응을 다룬다.

---

## 1. Activities, Components, and Costs

> Quantitative business case = 예상 benefit + 예상 cost. 첫 iteration은 DQ 전략 요소(tool 등)를 처음 구축하므로 비용이 큼.

### 단계별 활동 (data quality improvement cycle 기반)
- Business case → Data discovery(Ch5) → Rule development+Monitoring(Ch6/7, =DQ tool 구현: design·build·test·go-live) → Remediation(Ch8) → Embedding BAU(Ch9).
- Waterfall vs Agile: 활동은 broadly 동일(plan·tool 선정·data 연결·rule/report 설계·구현·test). 규제 산업은 문서 요구↑ (예: pharma GxP assessment).

### Early phases
- DQ manager(data management group 소속)가 주도. 성숙도 낮은 조직엔 역할 부재 → 우선 1명 추가 비용만 충당하는 simple business case 권장.
- **High level scoping**: 이론상 전 부서 RCA(process/tech/data 이슈 분류)지만, 실무는 비현실적(시간·우선순위·정치). **"push on open doors"** — 이미 DQ 문제를 인지하고 도움을 찾는 stakeholder 공략. 사례: supply chain 리더는 거부(third-party generic cleanse 실패), CFO가 champion이 되어 showcase → 이후 supply chain도 참여.

---

## 2. Cost Modeling

> 대부분 비용은 people cost. 그러나 non-people cost도 고려해야 완성.

### People cost
- 역할별 phase별 FTE effort 모델링. role 목록·일수·rate(예: 개발자 \$400/day) → 비용 산출.
- **Deliverable 기반 sense-check (Fig 3.6)**:
  - 고정: tool setup ~100 days, report ~10개 × 25 days/report.
  - 변동: rule 수(low/med/high complexity별 표준 일수), system당 ETL ~80 days.
  - 예: 150 rules × 3 systems = 1,070 days ≈ 계획 1,062 days → 추정 견고.

### Non-people cost (Table 3.2)
- **Tools**: DQ tool(Informatica DQ, SAP Information Steward, Semarchy — 신규 도입 가능성 높음), ETL tool(보통 기존 보유), visualization(Power BI·Tableau·Qlik — 보통 기존 보유).
- **Infrastructure**: cloud/hardware capacity, network capacity.
- **3rd-party support**: BAU 이관 후 application maintenance(Capgemini·IBM·Cognizant).
- Tip: 기존 vendor portfolio 활용(예: SAP Information Steward는 Data Services Enterprise Edition의 일부) → license 전환이 신규보다 저렴.

---

## 3. Quantitative Benefit Estimates (3 접근법)

> Benefit 정량화가 어려운 이유: business case 단계엔 rule이 없어 문제 규모 미상 + benefit이 "one step removed"(데이터 수정 자체가 아닌, 사용·의사결정 시점에 발생).

### 예시의 어려움
- supplier remittance email 누락 → finance팀이 ERP "고장"으로 오인. benefit = 결제 문의 감소 효과. 정확한 예측은 invoice량·문의량·80% 감소 가정 등 다단계 계산 필요. rule 1개당 이 정도 → 100+ rule이면 계산이 구현보다 더 오래 걸림.

### Approach 1 — payback 충당 계산
- 알려진 이슈를 우선순위대로 정확히 계산, 비용을 짧은 기간(예: 2년)에 회수할 때까지 누적. 좋은 후보: master data 이슈(다수 transaction 영향), revenue 영향, 프로젝트 지연, 규제 영역.
- **Pro**: 표준 관행, 대부분 조직이 인정. **Con**: 두드러지지 않음(payback 길 수 있음), 소규모 조직은 충당 어려움, 가장 시간 소모적.

### Approach 2 — 제한적 계산 후 extrapolate
- 2~3개 핵심 이슈만 deep dive, 나머지 모집단에 외삽. 계산 소요 시간 기록(예: 3 rule = 2주 → 150 rule = 100주, 비현실적임을 설명).
- 비용을 "target"으로 두고 반박 어려운 가정 제시(예: 비용 \$487k, 3개로 \$150k(31%), 나머지 30개가 평균 \$11,250만 내도 회수).
- **Pro**: approach 1보다 효율적이면서 개별 계산은 견고. **Con**: 전통 방법론과 불일치, 도전받기 쉬움. 소규모/유연 조직에 적합.

### Approach 3 — top-down benchmarking
- 조직 metric을 유사 조직 benchmark(Gartner·Hackett Group)와 비교 → gap의 benefit 추정 → 설문으로 DQ 기여 % 산정.
- 예: P2P cost per invoice \$35→\$15, gap × invoice량. 운영자 설문: DQ 20%/system 40%/process 30%/other 10% → benefit × 20%.
- **Pro**: 리더 전략과 직결, 효율적. **Con**: 정치적(인력 감축 함의), 상세 분석 부족으로 도전받기 쉬움. 강한 리더 backing이 있을 때 적합.

---

## 4. Qualitative Benefits (risk avoidance)

> 정성적 benefit은 대부분 risk 회피 — 확률적이라 정량화 어려움. compliance·reputation·employee·future project risk.

- 단순 나열·generic backup은 무시됨 → **survey·focus group으로 정량 요소 부여**(예: "76%가 poor DQ가 규제 준수에 영향" 등 soundbite).
- 함수·seniority별 grouping → 리더의 의견과 운영진 의견의 disconnect를 "reality check"로 활용.
- Table 3.4 예: compliance(76%), employee engagement(P2P churn +47%, 82%), 의사결정(senior 43%/operations·commerce 72%, loss-making 계약 사례), reputation(NPS 20→-10, 64%), future project(chart of accounts 91%).

---

## 5. Anticipating Leadership Challenges

> Board는 한정 budget으로 challenging question을 던진다. 사전 준비·1:1 사전 설명이 핵심.

### "Excel will do the job"
- 대응: 단일 rule·단일 시점이 아닌 **portfolio of rules**의 holistic·자동·일일 갱신·trend. Excel 보관은 GDPR/손상 위험. bespoke failed-data view 제공. 100+ rule 일일 분석은 Excel로 비현실적.

### Ownership of ongoing costs
- tool 유지·license·report 운영 비용의 미래 "home" 필요. 함수별 분담은 실패하는 편 → **사전에 단일 함수 리더가 운영 budget 부담 합의**(data office / IT / DQ lead 소속 함수(흔히 finance)).

### Excessive cost
- 각 비용 항목별 예상 challenge·답변 문서화(예: testing은 2단계+defect 수정 포함). 축소 scope 버전 준비. 첫 이니셔티브가 가장 비싸다(고정 setup). 미실행 시 비용도 제시.

### "Why a DQ tool?"
- DQ tool은 visualization보다 비싸 보임(Power BI \$9.99/user). 대응: 타사 reference call, tool 없이 시작해 가치 입증 후 자금 확보 가능하나 **장기 가치 극대화엔 purpose-built DQ tool 필수**(MDM·visualization tool 전용 X).

---

## Summary (핵심 정리)

- DQ business case는 성공만큼 실패도 많은 단계 → 철저한 준비와 승인 board 멤버와의 사전 공유가 critical.
- DQ business case는 전형적 형태와 달라 처음엔 경쟁력이 낮아 보일 수 있으므로, 1:1로 challenge에 미리 답하는 것이 유리.
- 승인 직후 몇 주가 매우 중요 → 다음 챕터(4장)는 이니셔티브를 성공적으로 set up하는 그 기간을 다룬다.
