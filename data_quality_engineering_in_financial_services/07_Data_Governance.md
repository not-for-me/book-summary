# 07. Data Governance

## 챕터 개요 (3줄 요약)

- Data governance는 데이터의 definition·relationship·availability·stewardship·ownership·quality·integrity·security를 포괄·규율하는 관리 체계다.
- Governance(전략·정책 framework)와 Management(정책 실행: architecture·practice·procedure)를 명확히 구분하고, centralized/distributed/hybrid 조직 모델을 제시한다.
- Council·Data Management·Business function 협업으로 DQS·data dictionary·RACI·scorecard를 정의하며, data management maturity model로 성숙도를 평가한다.

---

## 1. Establishing a Data Governance Function

> Governance의 핵심 목표: 데이터가 이해되고, quality가 측정·검증되며, 사용 준비(certified)가 확인되는 것.

- 정의: 데이터의 정의·관계·가용성·stewardship·ownership·quality·fitness·integrity·security의 포괄적·규율적 관리.
- 시작점: data asset의 owner·steward·custodian 정의 → authority·accountability 정책 수립. cybersecurity·data protection은 InfoSec과 공동.
- 조직 형태: centralized / decentralized / hybrid 가능.

### 일반 원칙

- 데이터와 그 가치를 전사적으로 이해
- 데이터를 function에 fit-for-purpose하게 설계·통합
- Data quality를 정의·정량화·측정·benchmark
- Quality metric·scorecard로 의사결정 지원
- Stewardship 역할·책임을 명확히 조직

---

## 2. Governance vs Management

> Governance는 strategy·objective·policy의 framework, Management는 그 정책을 실행하는 architecture·practice·procedure.

```text
Data Governance function           Data Management function
- 정책/표준/통제 operating model     - data 관리·control 운영 실행
- Data Management 감독              - data steward로 quality·provisioning 책임
- DQS 기준 quality metric 모니터링    - DQS 기반 statistics·quality metric 생성
```

### Governance 조직 모델

```text
Centralized  단일 governance, 정책·통제·결정권 집중   → 대형·계층 조직에 적합
Distributed  business별 다수 governance, 동등 권한    → 고자율 business에 적합
Hybrid       중앙 정책 + business별 localized 결정권   → 협업형 조직에 적합
```

---

## 3. Creating a Data Governance Program

> 협업적 engagement로 Data Governance function(center of excellence)을 세우고, council이 workstream을 구동한다.

### Data Governance Council 주요 workstream

- Data element 매핑 & data dictionary 개발
- DQS·metric·KPI 정의
- Data management 표준·usage 정책 정의
- Quality 측정·평가 방법 정의
- Training·messaging·progress reporting (정성 + 정량 + 운영 지표)

### 기능별 역할

- **Data Management**: data store·element inventory, RACI matrix, DQS 정의, 8개 dimension 기반 quality scorecard, fit-for-purpose score, IT와 architecture 개선 협업.
- **Business functions**: data usage·ownership 정책 정의, information sensitivity 분류(unrestricted/confidential/restricted)로 access control 정렬, capture·validation 표준, retention·archiving 정렬.

### Enhanced operating model 특징

- 활발한 governance community, 정의된 steward·owner, data dictionary·inventory, dashboard, 정성/정량 측정의 지속 refine, governance가 business 의사결정에 기여.

---

## 4. Business Value & Deliverables

> 정량적 측정이 정성적 의견보다 business value 인식에 효과적 (예: quality report로 완전성·정확성 입증 + error 감소 추적).

### 주요 deliverable

```text
Policies/standards  → usage 정책, data ownership matrix, 정의된 역할
DQS                 → 8개 dimension validation, quality metric
Quality metrics     → 측정·validation, fit-for-purpose, business impact score
Process/app changes → architecture 재설계, data flow 개선
Data inventory      → data dictionary, metadata
```

### Driver → Business value

- **Confidence 증가**: noise 감소, approved data 사용, 분석 신뢰도·정확도 향상.
- **Cost reduction**: 중복 처리 제거, reconcile·revalidate 낭비 감소, time-to-market 개선.
- **Data inventory 지식**: data dictionary, metadata, lineage, access/security.
- **Governance controls**: in-house 기술 우선·저비용 wireframe, 필요 시 vendor 검토.

---

## 5. Data Management Maturity

> 성숙도 모델로 운영·기술의 excellence와 개선 영역을 파악. 목표는 governance + 정량적 quality management의 통합/최적화 단계.

```text
Emergent      ad hoc, 반복 불가, 도구 제한적
Developing    lifecycle 기반 managed, event-driven triage
Structured    architecture·platform 기반 반복 가능, event-driven monitoring
Integrated/   정량 metric·feedback으로 최적화,
Optimized     center of excellence, DQS·통합 metric 사용
```

> EDM Council의 Data Management Capability Assessment Model 참조 권장.

---

## Summary (핵심 정리)

- Data Governance function의 핵심 목적은 데이터 asset의 관리·ownership·stewardship·access·use에 구조적·규율적 접근을 촉진하고 데이터 가치 인식을 높이는 것.
- Quality metric과 process metric을 활용해 데이터 관리 방법·process·architecture의 지속 개선을 견인하며, 모든 curation을 high-quality data 제공에 정렬.
- 다음 챕터들(8장~)은 thinking like a manufacturer·shape of data·DQS 개념으로 돌아가 governance 목표를 뒷받침하는 master data management 등을 다룬다.
