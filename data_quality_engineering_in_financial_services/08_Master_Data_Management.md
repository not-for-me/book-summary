# 08. Master Data Management

## 챕터 개요 (3줄 요약)

- MDM은 critical data를 정의·관리하는 architecture·standard·process·policy·tool로, validated·approved·certified된 단일 mastered volume을 제공한다.
- 데이터는 raw → staged → mastered 3단계를 거치며, staging에서 DQS·anomaly detection·remediation을 수행한 뒤 master로 승격된다.
- Data governance + MDM + EDM이 stack으로 정렬되어 data-quality-first 문화와 proactive validation을 구현한다.

---

## 1. Mastering Data (raw → staged → mastered)

> Manufacturing처럼 proactive validation(사용 전 검증)이 reactive reconciliation보다 downstream 영향을 줄인다.

### 3가지 데이터 상태

```text
Raw      vendor/내부 app에서 받은 untouched 초기 데이터
Staged   transformation·enrichment·cohesion 평가 + DQS 기반 quality 평가·
         anomaly detection·remediation, quality metric 생성
Mastered DQS 충족·검증·approved → master volume으로 promote·release
```

- Master 데이터는 domain별 volume(security, price, account, holdings 등)으로 조직. 거대 중앙 단일 store는 best practice 아님 — 각 collection이 고유한 architecture·quality·retention·access 요구.
- 기술 선택(DB, file system, Python/Excel 등)은 firm 재량.
- 중복 revalidation·reconciliation이 여러 function에서 반복되는 것이 operational scale의 최대 장벽.

### Proactive validation 이점

- 중복 revalidation 제거로 효율↑, 비용↓, risk↓, IT delivery 효과↑, 데이터 신뢰↑, business 유연성·혁신.

---

## 2. MDM Stack (Governance / EDM / MDM 정렬)

> Data governance가 모든 MDM layer를 가로지르고, EDM이 정의·inventory·ownership·stewardship·quality assurance에 집중.

```text
[Data Governance]  전 layer span: 정책·표준·leadership·project methodology·training
[MDM 계층]
  Data access/distribution  role/policy 기반 access, InfoSec, abstraction(API·web service)
  Data quality assurance    DQS, 정량 validation, anomaly detection·remediation, metric
  Data inventory            dictionary·glossary·lineage
  Physical structures       data architecture(raw/staged/mastered), logical modeling
  Information Technology     DB·schema·table
[EDM]  data management·control, quality expertise, use policy, ownership/stewardship
```

- **Data abstraction layer**(custom API, web service, DB view): consumer를 physical infra로부터 격리 → 일관된 provisioning + IT의 구조 변경 유연성.
- **Data glossary**: data element·정의·metadata의 검색 가능 collection. **Data inventory**: 물리 volume·data class·source·structure·record count 등.
- **CDE(Critical Data Element)**: 최고 business value·impact. 오류 시 regulator/client/auditor misrepresentation, 잘못된 의사결정.

---

## 3. Governance & Management Synergies

> Governance와 MDM은 상호보완적으로 high-quality data 제공을 견인한다.

- **Data Governance**: data asset 정의·authoritative 정보, access·use 가이드, steward·owner의 standard of care, quality metric, data criticality 합의. metadata·data type·naming 표준 촉진, DQS 기준 quality 모니터링, data-quality-first 교육.
- **EDM function**: 여러 data class별 steward 팀. master architecture로 DQS 대비 quality 평가, remediation 수행 권한, data inventory(dictionary·glossary·lineage) 관리.
  - **Data owner**: 최고 SME, source·methodology·use 결정권, DQS 정의.
  - **Data steward**: acquisition·profiling·validation·remediation·metric·preservation·distribution curation. centralized center of excellence로 조직화 추세.

---

## Summary (핵심 정리)

- 기술·구조보다 중요한 것은 consumer DQS를 충족하는 curated high-quality data. raw는 staging에서 validation·enrichment·transformation을 거쳐 master volume의 shape에 정렬된다.
- Mastering의 목적은 release 전 안전한 환경에서 quality·cohesion·collection을 검증하고, class별 master volume(security, reference, account, holdings)으로 조직하는 것.
- DQS validation에는 Excel(소량)·Python(대량)·third-party EDM app·자체 구조 등 다양한 기술 사용 가능.
- 다음 챕터(9장)는 data definition·integrity·management 활동을 담은 project development methodology를 다룬다.
