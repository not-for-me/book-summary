# 09. Data Project Methodology

## 챕터 개요 (3줄 요약)

- 요구사항부터 구현까지 data-intensive 프로젝트 lifecycle의 task·artifact를 data definition → data integrity → data management 3단계로 정리한다.
- Waterfall·Agile·Kanban 어느 방법론에도 통합 가능하며, business·technology·data governance 요구사항을 각각 분석한다.
- 핵심 산출물(data dictionary·data flow·DQS·validation program·scorecard·RACI matrix)로 data pipeline에 quality를 engineering한다.

---

## 1. 방법론 개요 (3단계 질문)

> 각 단계는 핵심 질문에 답한다.

```text
Data Definition  : 어떤 데이터가 어떤 business 목적으로 필요한가?
Data Integrity   : business의 data quality 요구는 무엇인가?
Data Management  : 구현 후 DQS·기대를 보장할 support model은?
```

---

## 2. Business Requirements

> 금융업 대부분 프로젝트는 data component를 가짐. "I assumed…"가 rework·불만을 부른다 → 먼저 business use case 정의.

- **Business use case**: process와 data source·volume·distribution 기술.
- **Business process map / flow diagram**: current·future state. future-state는 data control point·transformation을 표시 → pre-use validation·DQS의 기초. upstream/downstream dependency 문서화 필수.
- **Impact analysis** (CDE 중심):
  - **High**: misrepresentation·잘못된 의사결정 → 벌금·신뢰 상실·규제 위반. process failure 상태, 즉시 remediation.
  - **Medium**: 효율 저하(자원 시간 낭비). 운영은 가능하나 impaired, triage 필요.
  - **Low**: 경미, 운영 가능. correction 우선순위 낮을 수 있음.
- **Data quality scorecard**: 프로젝트 초기에 정의, 후반에 상세 개발.
- **Data usage policy**: 데이터 source·use·storage·distribution의 승인 기준 (architecture, access, naming, steward 등).

---

## 3. Technology Requirements

> 데이터는 technology 안에 존재 → application data processing use case와 flow가 DQS 개발을 informing.

- **Application data processing use case**: app의 architecture·source·volume·processing·distribution 기술 세부.
- **Application process map / data flow diagram**: data source·inbound/outbound flow·processing function, data type·format·structure 등 기술 세부 (business diagram에 없는 reference data·cohesion 요구도 드러남).

---

## 4. Data Governance Requirements (3 task 카테고리)

### Data Definition Tasks
- Data element·collection·cohesion 정의, data dictionary 개발.
- **Data model**: data type·size·mandatory/optional·precision·PK/FK 관계.
- **Data lifecycle**: origination→processing→use→transformation→storage→retention→retirement.
- **Data flows / transformations / distributions**: translation(code), calculation, derivation(algorithm), augmentation(concatenation); query·API·file 제공.

### Data Integrity Tasks
- **DQS 정의**: validation·verification control과 RACI의 기초 (가장 중요한 artifact).
- **Data quality assessment**: 기존 데이터의 fit-for-purpose 재검증 (source 데이터가 새 요구를 충족 못하는 경우 多 → gap 발견 시 remediation·control 추가).
- **Data realignment/remediation**: steward가 augmentation·cleanup·pre-use validation 구현.
- **Quality controls (validation vs verification)**:

```text
Data validation   = primary, PRE-use control  → 사용/배포 전 datum이 DQS 충족 확인
Data verification = secondary, POST-use control → 수신 데이터가 여전히 DQS 충족 확인
→ anomaly 발생 시 데이터+기술(query·API·전송 logic) 양쪽 triage 가능
```

- **Measuring & scorecarding**: V/S/IV 계층 시각화로 fit-for-purpose·historical pattern·tolerance·통계 제시.
- **Data integrity sensors**: 제조업 sensor처럼 동작하는 validation program — pre-use validation, post-use verification, status monitoring, quality alert.
- **Access controls**: need-to-know 기반 provisioning, InfoSec 정책 준수.

### Data Management Tasks
- **Roles**:

```text
Data owner     최고 SME·직접 사용자, DQS 정의, source·use 결정권 (데이터 관리는 안 함, "client")
Data steward   curation 책임(acquisition·profiling·validation·remediation·metric·distribution)
Data custodian IT — 안전·고성능 infra 제공 (데이터 무결성 책임은 아님, 기술 무결성 책임)
```

- **RACI matrix**: data quality 이슈 대응의 Responsible/Accountable/Consulted/Informed 명확화.

```text
예) Empty account number  → Completeness alert → R: Client Service, A: Data Management
    Unknown transaction   → Conformity alert   → R: Operations,    A: Data Management
    Incorrect market value→ Congruence alert   → R/A: Operations (+Trading·Compliance informed)
```

- **Tool certification**: 새 data class·validation logic이 기존 tool에서 지원되는지 검증. RACI를 test case 기반으로 활용. 미지원 시 IT가 backup(비이상적·자원 낭비).

---

## Summary (핵심 정리)

- 일반 개발 방법론이 기술·infra에 집중하는 것과 달리, data project methodology는 프로젝트 범위 내 모든 데이터 측면에 집중한다.
- 핵심 artifact: data dictionary, data flow, DQS, validation program, scorecard, RACI matrix — 이것들이 data architecture·pipeline에 quality를 engineering하는 열쇠.
- 복잡도 가이드: **simple**(<50 elements, 기존 well-understood) vs **complex**(>50 elements, 다중 복잡 volume·신규 pipeline). task별 abbreviated(○) vs full(●) 상세 수준 권장.
- 다음 챕터(10장)는 모든 구성요소를 enterprise data management로 통합한다.
