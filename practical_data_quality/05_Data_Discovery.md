# 05. Data Discovery

## 챕터 개요 (3줄 요약)

- Data discovery는 어떤 데이터가 가장 중요한지 파악하고 challenge를 식별해 이니셔티브 scope과 data quality rule을 정의하는 과정이다.
- business strategy → objective → challenge → process/analytics/data로 연결하는 hierarchy를 따라 우선순위를 정한다.
- Data profiling(string·pattern·field value·completeness·uniqueness/distinct)으로 데이터의 "extremity"를 드러내 초기 rule을 도출한다.

---

## 1. Discovery Process 개요

> 가장 큰 실수는 잘못된 데이터에 집중하는 것. critical process·의사결정에 영향 없는 데이터를 고치면 이니셔티브가 인정받지 못해 funding을 잃을 수 있다.

- 출발점: 조직 strategy·stakeholder objective·달성을 막는 challenge 이해. stakeholder가 "data 관련"으로 필터링하지 않고 holistic하게 말하도록 유도.
- challenge → process(root cause)·system(profiling 대상)·data(table/field) 연결.
- 사례: "pay immediately" supplier 없다던 process owner — profiling 결과 40개 supplier(수백만 달러)가 즉시 결제로 설정됨.

---

## 2. Understanding Strategy, Objectives, Challenges

> 잘못된 배경의 sponsor가 익숙한 데이터(예: 구매 출신 → supplier data)에 집중하다 실패. 사례: services 조직에선 client data가 우선인데 supplier에 집중.

### Stakeholder 식별 (Table 5.1)
- central strategy team, Data Governance Council(owner/steward), 조직도, data/analytics team(BI·data science의 report·lineage).

### Engagement 어려움
- 고위 임원 일정 확보: 이메일에 그들 영역의 의미있는 DQ 예시 포함, CDO가 발송, "맥락 제공+적임자 안내"임을 명시.

### Conversation 구조
- **Initial**: 이니셔티브 맥락, 영역 overview, 관련 strategic pillar, 현재·미래 challenge (data로 한정 X).
  - 사례: HR 리더가 "data와 무관"이라던 bonus 프로세스(3개월→1개월 목표) — 실제론 10%+ line manager가 중복·필수필드 누락으로 helpdesk ticket. 데이터 개선만으로 목표 달성(신규 시스템 불필요).
- **Detailed** (한 단계 아래 운영진): challenge를 질문·DQ implication으로 변환·검증 (Table 5.3 예: 수요 예측 불가 → siloed/incomplete forecast data).
- 운영 최하단 reality check, IT application leader·system owner 조기 engage.

---

## 3. Hierarchy: Strategy → Process → Data

> challenge가 너무 많으면 strategy로 우선순위화. 더 많은 rule≠더 높은 우선순위.

```text
Pillar별 평가 (Fig 5.2): rule 수 × 구현 complexity
  복잡도 요인: system 수, rule 복잡도, stakeholder 불명확, 부서별 상이 운영
  → 단순히 rule 多·complexity 低 pillar 선택은 함정
  → benefit·pillar 간 dependency 고려 (예: 신제품 출시 pillar가 commercial pillar의 전제)
```

- de-scope되는 pillar의 기대 관리 필수 → discovery 미팅 초기 맥락에 포함, scoping 결정은 전체 합의.

### Challenge ↔ Process/Data/Reporting 연결 (Table 5.5)
- 각 challenge별 파악: 영향 process·팀·step, 해결 시 개선점, 현 workaround, system, table/field, 데이터 subset, source, **active 데이터 식별법**(비활성 포함 시 무의미한 failure 양산), duplication 식별법.
- 부서 간 책임 전가 흔함(finance vs procurement) → DQ팀이 cross-functional 중재. 예: supplier bank details는 procurement가 수집하나 finance가 사용.
- table/field 미상 시 translation 팀 활용: system center of excellence, operational excellence(ARIS), analytics(data engineer), metadata solution(business 개념↔system field).
- **Reporting 영향**도 점검: manual workaround(latency·human error 위험), 미제공/지연 reporting 요구.

---

## 4. Data Profiling

> 데이터셋을 평가해 값·string 길이·completeness·분포 패턴을 컬럼별로 제공. Excel 필터링도 기초적 profiling.

### 도구
- Informatica, Ataccama, IBM(Watson/InfoSphere/Match 360), SAP Information Steward, Talend. (저자는 Informatica·SAP 구현 경험.)
- tool 없이도 가능: Power BI Desktop(무료)의 Column quality/profile/distribution (전체 dataset 옵션).

### 주요 capability
```text
String evaluation   컬럼별 min/max/mean/median 문자열 길이
Field value eval    컬럼별 min/max/mean/median 값 (dimension table에 유용)
Pattern matching    데이터 타입 패턴 (예: UK 우편번호 XX11 1XX), 패턴별 record 수
Completeness        null / blank / zero 개수 (셋은 기술적으로 다름)
Distinct            적어도 1개 record를 가진 값의 수
Uniqueness          정확히 1개 record만 가진 값의 수
```

### 활용 예시
- **String/pattern (UK VAT)**: 길이 9(GB 생략)·14("Not Registered")·11(정상)·10/12/13(오류). 맥락 확인 후 rule 도출 → 11자는 GB111111111 패턴, 14자는 "Not Registered". **profile은 UK로 filter할 때만 유효**(국가별 형식 상이).
- **Field value (UK 도로번호)**: min 0/max 9999는 실재 안 함. 0이 27% → 의미 있음(번호 없는 도로). road type과 correlate해 rule 도출.
- **Completeness (날씨)**: null ~2% + "Other"·시간값(17:00) 등 무의미값 합산. subset filter 필요(예: micro-entity는 VAT 50%, 대기업 90%).
- **Uniqueness/distinct (Accident_Index)**: 고유해야 할 index가 36% 중복(349개 값에 분포, 한 값에 7000+ row) → 동일 사고가 여러 위치에? 심각한 데이터 이슈, source 재확인 필요.

---

## 5. Connecting to Data

> File 업로드보다 database 직접 연결이 우월 — 파라미터 바꿔 재profiling 용이, source에 가까워 ETL 변형 회피, 본 이니셔티브에도 필요.

- 고려사항: owner 연결 허가, DB 성능 영향, data privacy. 필요 시 ETL로 OLAP DB에 추출(업무 외 시간), 또는 대표성 있는 test system 연결(governance 낮음).
- 승인 지연 시 file 추출이 빠를 수 있으나 data security팀과 사전 협의.

---

## Summary (핵심 정리)

- business strategy를 제대로 이해하면 조직이 이니셔티브가 우선순위를 안다고 신뢰하게 된다.
- discovery 정보로 challenge의 root cause를 조사하고 process·analytics·data에 연결해 어떤 데이터를 profile할지 결정한다.
- profiling은 매일 데이터를 쓰는 사람조차 놀랄 발견을 주며, 이제 data quality rule을 완전히 개발할 단계 → 다음 챕터(6장).
