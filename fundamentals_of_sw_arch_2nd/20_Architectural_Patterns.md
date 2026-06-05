# 20. Architectural Patterns

## 챕터 개요 (3줄 요약)

- 아키텍처 패턴은 디자인 패턴에서 영감받은, 문제에 대한 맥락화된 해법으로 스타일과 구분된다.
- 재사용(Hexagonal, Service Mesh), 통신(오케스트레이션/코레오그래피, CQRS), 인프라(Broker) 패턴을 다룬다.
- 패턴을 먼저 식별한 뒤 적절한 구현을 고르며, '베스트 프랙티스'와 구분해 트레이드오프를 분석해야 한다.

---

## 1. 패턴 vs 스타일 vs 베스트 프랙티스

> 패턴은 맥락화된 해법이며, '베스트 프랙티스'(생각을 끄고 항상 같은 해법)와 구분해야 한다.

도구·프레임워크는 패턴을 다양한 충실도로 캡슐화한다. 적절한 패턴을 먼저 식별하고 그다음 구현을 선택하라.

---

## 2. Reuse (재사용 패턴)

> 분산 아키텍처에서 도메인 결합과 운영 결합을 분리하는 것이 흔한 관심사다.

microservices는 "재사용보다 중복"을 택해 도메인은 디커플링하지만, 모니터링·로깅·인증·서킷브레이커 같은 운영 역량은 결합이 유리하다(팀별 관리는 혼란).

### Hexagonal Architecture (육각형 = Ports and Adapters)

도메인 로직이 중심, 포트·어댑터가 생태계와 연결. Alistair Cockburn 창안(이름 후회). 결함: DB를 단순 어댑터로 취급해 데이터 스키마를 비즈니스 로직과 분리 — microservices 원칙 위반. Eric Evans가 DDD에서 스키마도 비즈니스 로직 반영해야 한다고 교정. "도메인-운영 분리"의 약칭으로는 OK.

```
        +-- adapter --+
   port |   DOMAIN    | port
        |   LOGIC     |
        +-- adapter --+
   (DB is just another adapter <- the flaw)
```

### Service Mesh

Sidecar 패턴(직교 재사용)으로 운영 관심사를 도메인을 가로지르는 일관 계층에 격리. Orthogonal coupling(직교 결합): 모니터링처럼 도메인과 독립적이나 교차해야 하는 관심사. 트레이드오프: 일관된 격리 결합·인프라 조정 vs 플랫폼별 sidecar 구현·복잡화·팀 간 드리프트. Hexagonal은 범용, Service Mesh는 microservices에 적합 — 둘 다 '분리'라는 패턴의 구현.

---

## 3. Communication (통신 패턴)

> EDA에서 유래해 메시지·이벤트로 통신하는 모든 분산 아키텍처에 적용된다.

### Orchestration vs Choreography

- Orchestration(오케스트레이션): 코디네이터 서비스. 장점: 중앙 워크플로우·오류 처리·복구성·상태 관리. 단점: 응답성 병목·단일 장애점·확장성 저하·강결합.
- Choreography(코레오그래피): 코디네이터 없음. 장점: 응답성(병렬)·확장성·내결함성·디커플링. 단점: 분산 워크플로우 관리·상태 관리·오류 처리·복구 곤란.

```
  Orchestration:  [Orchestrator] -> A,B,C,D
  Choreography:   A -> B -> C -> D (no central coordinator)
```

### CQRS (Command Query Responsibility Segregation)

읽기와 쓰기를 분리. 쓰기는 한 데이터스토어로, (보통 비동기로) 읽기용 DB에 동기화. 읽기/쓰기 볼륨 차이가 크거나 보안 격리가 필요할 때 유용. 데이터 종류별로 다른 특성·데이터 모델 적용 가능.

```
  Client/Server:  App <--> DB (read+write)
  CQRS:  App -write-> [Write DB] --sync--> [Read DB] -read-> App
```

---

## 4. Infrastructure (인프라 패턴)

> 아키텍트는 컴포넌트·데이터·API뿐 아니라 인프라 간 결합도 신경 쓴다.

### Broker Patterns (브로커 패턴)

EDA에서 토픽/큐는 보통 발신자가 소유. Single-Broker(단일 브로커): 중앙 디스커버리·최소 인프라·단일 로깅 장소. 단 단일 장애점·처리량 한계. Domain-Broker(도메인 브로커): 관련 서비스 그룹별 브로커 공유(도메인 분할 반영). 더 나은 격리·확장성·내결함성. 단 디스커버리 어려움·인프라 비용·유지 부담. 둘 다 베스트 프랙티스 아님 — 디스커버리와 도메인 격리를 균형.

```
  Single-Broker:  all services -> [one broker]
  Domain-Broker:  domain group A -> [broker A]
                  domain group B -> [broker B]
```

---

## Summary (핵심 정리)

- 패턴은 맥락화된 해법으로 스타일·베스트 프랙티스와 구분되며, 패턴 식별 후 구현을 고른다.
- 재사용(Hexagonal/Service Mesh)으로 도메인-운영 결합을 분리하고, 직교 결합을 인식한다.
- 통신(오케스트레이션/코레오그래피, CQRS)과 인프라(Single/Domain-Broker) 패턴의 트레이드오프를 분석한다.
