# 03. Data Product-Centered Architectures

## 챕터 개요 (3줄 요약)
- 데이터 제품 중심 아키텍처는 사회적(조직) 아키텍처와 기술 아키텍처를 함께 설계해야 하는 소시오테크니컬(socio-technical) 시스템이다.
- VSM(Viable System Model)을 활용해 운영 평면(operational plane)과 관리 평면(management plane)의 핵심 기능과 역량을 정의한다.
- 데이터 메시, 데이터 패브릭, 데이터 센트릭/데이터웨어 등 대안적 접근법과의 관계 및 차이를 비교한다.

---

## 1. Designing a data-product-centric architecture
> 모놀리식에서 모듈형 데이터 관리로의 전환은 기술과 조직 아키텍처 양쪽 모두에 개입이 필요한 패러다임 전환이다.

- 시스템 아키텍처는 핵심 아키텍처(core architecture)와 메타 아키텍처(meta-architecture)로 나뉘며, 메타 아키텍처가 시스템의 진화 능력을 규율한다.
- 적응형 시스템에서는 진화를 규율하는 meta-architecture가 core architecture만큼, 혹은 더 중요하다.
- 아키텍처 결정은 "최후의 책임 순간(last responsible moment)"까지 미뤄 더 많은 정보와 성숙도를 확보한다.
- 데이터 관리 솔루션은 조직 전체 아키텍처의 일부이므로, 타 조직 성공 사례를 그대로 복사하는 것(cargo cult)은 통하지 않는다.
- 소시오테크니컬 시스템은 사회적 아키텍처(social)와 기술 아키텍처(technological)로 구성되며, 둘은 강하게 상호 연관된다.
- 정렬이 없으면 사회적 아키텍처가 기술 아키텍처를 결정한다(Conway's law); 역방향 설계(reverse Conway maneuver)도 위험하므로 두 아키텍처를 병행 설계한다.

### Architectural principles
- 모듈성(Modularity): 기술 모듈화 단위는 데이터 제품, 조직 모듈화 단위는 팀이며, 레이어가 아닌 기능별로 분할한다.
- 조합성(Composability): 모듈 간 상호운용성과 동적 역량(dynamic capabilities)을 통해 재사용·재구성을 가능하게 한다.
- 지속가능성(Sustainability): self-serve 플랫폼과 적합도 함수(fitness functions)로 품질·보안·건강성을 지속적으로 유지한다.

---

## 2. Dissecting the architecture's operational plane
> 운영 수준(System 1)에서 데이터 관리 기능이 갖춰야 할 4가지 핵심 역량을 정의한다.

- 데이터 제품 개발(data product development): 모듈형 솔루션을 구축하는 핵심 역량.
- 거버넌스 정책 수립(governance policy-making): 안전한 조합성을 위한 보안·운영·상호운용성 정책 정의.
- XOps(Extended Operations) 플랫폼 엔지니어링: 발행·검증·모니터링·비용통제·검색 등 공통 기능을 self-serve로 제공.
- 데이터 변환 활성화(data transformation enabling): 새 패러다임에 필요한 기술과 지식을 조직 전반에 전파.
- 각 역량별 예산을 분리 관리해야 데이터 제품 개발이 전체 예산을 잠식하는 리스크를 방지한다.

### VSM (Viable System Model)
```
 +---------------------------------------------+
 | System 5: Identity   (vision, values)       |
 | System 4: Intelligence (strategy, goals)    |
 | System 3: Control    (planning, priorities) |
 | System 2: Coordination (integration)        |
 | System 1: Operations (day-to-day execution) |
 +---------------------------------------------+
```

### XOps platform planes
```
 +-------------------------------+
 | Data Product Control Plane    |  -> lifecycle orchestration
 +-------------------------------+
 | Utility Plane                 |  -> abstraction over IT landscape
 +-------------------------------+
```

---

## 3. Dissecting the architecture's management plane
> 운영 역량(System 1)이 조직의 전략과 정렬되도록 관리하는 System 2~5의 책임을 분석한다.

- 정체성 시스템(Identity, System 5): 비전·가치·아키텍처 원칙을 정의하며 보통 연 1회 검토한다.
- 지능 시스템(Intelligence, System 4): 비전을 구체적 목표(goals)로 번역하며 약 6개월 주기로 검토한다.
- 통제 시스템(Control, System 3): 목표를 이니셔티브와 운영 활동으로 변환하고 우선순위·자원을 할당(분기/월 단위)한다.
- 조정 시스템(Coordination, System 2): 이니셔티브별 운영 활동을 조정하며 보통 주 단위로 검토한다.
- 운영 모델(operating model)은 관리 기능과 운영 역량의 상호작용 방식을 정의하는 핵심 설계 요소다.

---

## 4. Exploring alternative approaches to modern data management
> 데이터 제품 중심 접근법 외의 주요 현대적 데이터 관리 접근법 3가지를 비교한다.

- 데이터 메시(Data mesh): 도메인 오너십, 제품으로서의 데이터, self-serve 플랫폼, 연합 컴퓨팅 거버넌스의 4원칙 기반 탈중앙 접근법.
- 데이터 메시는 분석 목적에 특화되고 강한 탈중앙화를 전제하지만, 본서 접근법은 분석·비분석 모두 포괄하며 탈중앙화를 강제하지 않는다.
- 데이터 패브릭(Data fabric): 모든 데이터·메타데이터 처리와 지능형 자동화(knowledge graph 기반)에 집중하는 기술 중심 접근법.
- 데이터 패브릭은 XOps 플랫폼의 자연스러운 진화로 볼 수 있다.
- 데이터 센트릭(Data-centric)/데이터웨어(Dataware): 단일 중앙 데이터 모델(ontology + knowledge graph) 기반 모델 주도 접근법으로, 제품 중심 접근법과 병행 가능하다.

---

## Summary (핵심 정리)
- 데이터 관리 솔루션 아키텍처는 조직·기술 관점을 시너지 있게 동시 설계해야 하며, 만능 아키텍처는 존재하지 않는다.
- 모듈성·조합성·지속가능성 원칙과 VSM 기반의 운영/관리 기능 정의가 데이터 제품 중심 아키텍처의 뼈대를 이룬다.
- 데이터 메시·패브릭·센트릭/데이터웨어는 각각 다른 초점을 가지며 제품 중심 접근법과 보완적으로 결합될 수 있다.
