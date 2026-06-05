# 05. Designing and Implementing Data Products

## 챕터 개요 (3줄 요약)

- 데이터 제품이 지역(바운디드 컨텍스트) 및 전역 생태계에서 다른 제품·애플리케이션과 어떻게 상호작용하는지 설계한다.
- 데이터 제품의 메타데이터를 디스크립터 문서(DPDS, Data Product Descriptor Specification)로 기술하는 방법을 다룬다.
- 데이터의 소싱·처리·서빙 내부 컴포넌트 개발 시 고려할 핵심 원칙을 제시한다.

---

## 1. Designing data products and their interactions

> 데이터 제품은 자신이 속한 바운디드 컨텍스트(지역)와 조직 전체 생태계(전역) 모두와의 상호작용을 올바르게 설계해야 한다.

- 소스 정렬 제품은 운영 애플리케이션의 애그리거트(aggregate)를 읽어 재사용 가능하게 만들며 읽기 전용(read-only)이다.
- 소비자 정렬 제품은 소스 정렬 제품의 데이터를 보강(enrich)한다.
- 이벤트(event)는 불변(immutable)이며 상태(state)보다 정보량이 많아 변경 이력을 추적할 수 있다.
- 재사용 극대화를 위해 데이터를 상태와 이벤트 시퀀스 양쪽으로 노출하는 것이 좋다(복잡성 증가는 가치와 비교 평가).
- 컴포저블 아키텍처(Gartner): PBC(Packaged Business Capability)가 모듈화 단위이며 A-PBC(애플리케이션)와 D-PBC(데이터)로 나뉜다.

### Global ecosystem & context mapping
- 외부 시스템(COTS, Commercial Off-The-Shelf 소프트웨어, 레거시, 서드파티)은 유비쿼터스 언어를 쓰지 않아 어떤 컨텍스트에도 속하지 않는다.
- DDD(Domain-Driven Design)의 컨텍스트 매핑 패턴: OHS(Open Host Service), SK(Shared Kernel), CF(Conformist), ACL(Anti-Corruption Layer), PL(Published Language).
- OHS는 항상 구현 권장, SK는 절대 비권장, ACL은 외부 시스템 연동 시 항상 권장, PL은 전략적 자산(고객·계약·제품) 통합에 권장한다.

### Data product internals
- 데이터셋(dataset, 개념 모델)은 하나 이상의 데이터스토어(datastore)로 물리 구현되며 소비용·내부용으로 나뉜다.
- 소비자는 데이터스토어에 직접 접근하지 않고 출력 포트의 데이터 서비스(data service)를 통해 접근한다.
- 데이터 파이프라인(data pipeline)은 랜딩 데이터스토어에서 읽어 변환 후 소비용 데이터스토어에 저장하며, 어댑터(adapter)가 획득·재배포를 담당한다.

```
  input --> [adapter] --> landing store --> [data pipeline] --> consumption store --> [adapter] --> output
                                (business logic)       (hexagonal architecture)
```

---

## 2. Managing data product metadata

> 데이터 제품 디스크립터는 모든 컴포넌트를 기계 판독 가능하게 형식적으로 기술하며, 공개 부분이 데이터 계약(data contract)이다.

- 좋은 사양은 표현적(expressive)·인간/기계 판독 가능·유연·확장 가능·결합 가능·기술 독립적이어야 한다.
- 사양은 점진적으로(최소 필수 메타데이터부터) 확장하며, 각 메타데이터는 명확한 용도가 있어야 한다.
- 플랫폼은 디스크립터 없거나 유효하지 않으면 배포를 막고, 검색·외부 도구(데이터 카탈로그·마켓플레이스) 배포를 지원한다.

### Data Product Descriptor Specification (DPDS)
- DPDS(Open Data Mesh Initiative)는 JSON/YAML로 작성되며 OpenAPI와 구조가 유사하다.
- 최상위 구성: Info Object, Interface Components(포트), Internal Components(애플리케이션·인프라).
- 각 포트는 약속(promises), 기대(expectations), 의무(obligations) 세 블록으로 구성된다(Mark Burgess의 약속 이론 promise theory 기반).
- Standard Definition Object로 외부 사양(OpenAPI=REST, AsyncAPI=스트리밍, Datastore API=쿼리)을 참조한다.
- 신뢰성(trustworthiness)은 약속을 지키는 능력으로 측정되며 셀프서브 플랫폼이 측정한다.

---

## 3. Managing data product data

> 소싱·처리·서빙 컴포넌트는 헥사고날 원칙에 따라 비즈니스 로직을 입출력 방식과 분리해 구현한다.

- 소싱(sourcing): 푸시/팝(push/pop) 모드로 데이터를 획득하며, 어댑터가 ACL로 기술적 변환(복호화·구조 변환)만 수행한다.
- 트리거 정책은 시간 기반(time-based)과 이벤트 기반(event-based)이 있으며, 이벤트 트리거 팝 모드를 권장(의존성 감소).
- 외부 오케스트레이터가 아닌 소비자가 무엇을·언제 소비할지 결정하는 코레오그래피(choreography) 패턴을 선호한다.
- 처리(processing): 재현성(reproducibility)은 데이터 불변성과 이중시간성(bi-temporality: validity time + transaction time)으로 달성한다.
- 이중시간성 관리는 구체화(materialization)와 스냅샷팅(snapshotting)이 있으며, 현대 저장 시스템의 타임 트래블(time travel)이 복잡성을 가린다.
- 예측 가능성은 멱등성(idempotence), 버전 관리(versioning), 격리(isolation)로 달성한다.

### Serving data
- 소비 방식: 온라인 쿼리(Datastore API), 단건 접근(OpenAPI), 스트리밍(AsyncAPI).
- 출력 포트도 전용 어댑터로 구현하며, 플랫폼 수준 전송 계층(transport layer)이 라우팅·보안을 중재할 수 있다.
- 보존 정책(retention policy)은 디스크립터의 출력 포트 Promises Object에 명시한다.

---

## Summary (핵심 정리)

- 데이터 제품의 지역·전역 상호작용 설계는 제품 간 상호운용성과 팀 간 조정을 보장하므로 중요하다.
- 디스크립터 파일(DPDS 예시)은 메타데이터를 균일하게 관리해 발견·이해·생애주기 자동화를 촉진한다.
- 소싱·처리·서빙 내부 컴포넌트는 비규범적이나, 멱등성·불변성·격리 등의 핵심 원칙을 따라 관리·사용 용이성을 높인다.
