# 07. The Scope of Architectural Characteristics

## 챕터 개요 (3줄 요약)

- 현대 아키텍처는 시스템 전체가 아닌 서비스 단위로 서로 다른 특성을 가질 수 있어 '범위(scope)' 개념이 중요하다.
- 아키텍처 퀀텀(architecture quantum)을 특성의 범위·모듈성을 측정하는 단위로 정의한다.
- 특성 범위(퀀텀 경계)가 아키텍처 스타일·서비스 입도·클라우드 선택에 어떻게 영향을 주는지 설명한다.

---

## 1. Architectural Quanta and Granularity (아키텍처 퀀텀과 입도)

> 아키텍처 퀀텀은 독립 배포되고 높은 기능적 응집도를 가지며 외부 정적 결합이 낮은, 한 세트의 특성 범위를 정하는 단위다.

퀀텀(quantum, 복수형 quanta)은 "독립적으로 실행되는 시스템의 최소 부분"이다. 네 가지 속성: (1) 한 세트의 아키텍처 특성 범위를 설정, (2) 독립 배포 가능(데이터베이스 포함 — 단일 DB 레거시는 quantum=1), (3) 높은 기능적 응집도(bounded context와 매칭), (4) 외부 구현 정적 결합 낮음, (5) 다른 퀀텀과의 동기 통신 고려.

```
   Monolith + single DB  =  one quantum
   Microservices:
     [Svc A + DB-A]  [Svc B + DB-B]  [Svc C + DB-C]
       quantum 1       quantum 2       quantum 3
```

### DDD Bounded Context (도메인 주도 설계의 경계 컨텍스트)

Eric Evans의 DDD(Domain-Driven Design)는 복잡한 도메인을 bounded context로 분해한다. 전사 공유 Customer 클래스 대신 각 도메인이 자체 Customer를 두고 통신 지점에서 차이를 조정한다(공유로 인한 강결합 회피).

### 결합의 유형

Semantic coupling(의미적): 문제 자체의 자연스러운 결합(주문→재고·고객·판매). Implementation coupling(구현적): 단일/분산 DB 등 구현 결정. Static coupling(정적): 서비스 간 '배선' — 같은 결합 지점(예: 공유 DB)에 의존하면 같은 퀀텀. Dynamic coupling(동적): 퀀텀 간 통신 시 작용하는 힘. (팁: 범위가 좁을수록 강결합 허용, 넓을수록 느슨한 결합이 필요.)

---

## 2. Synchronous Communication (동기 통신)

> 동기 통신은 분산 아키텍처에서 서로 다른 운영 특성을 가진 퀀텀 간에 타이밍·블로킹 문제를 일으킨다.

예: Auction 서비스가 Payment 서비스를 동기 호출하는데 Payment가 500ms당 1건만 처리 가능하면, 경매 다수 종료 시 호출이 실패한다. 비동기(메시지 큐)는 큐가 버퍼 역할을 해 충격이 덜하지만, 지속 과부하 시 큐가 넘친다. 동기 통신은 특성이 다른 부분 간에 매우 비관용적이다.

---

## 3. The Impact of Scoping (범위 설정의 영향)

> 특성 범위(퀀텀 경계)는 적절한 아키텍처 스타일과 서비스 경계 선택을 돕는다.

단일 특성 세트로 충분하면 모놀리식, 여러 세트가 필요하면 분산 아키텍처를 택한다. 분산이면 퀀텀 경계→영속성(모놀리식=단일 DB, 분산=단일 DB 또는 서비스별 분할)→통신 방식(동기/비동기)을 차례로 결정한다.

### Kata: Going Green

중고 전자기기 재활용·재판매 업체 GG 사례. 특성 분석 시 세 클러스터가 형성된다: 공개 부분(확장성·가용성·민첩성), 백오피스(보안·데이터 무결성·감사가능성), 평가(assessment) 부분(유지보수성·배포성·테스트성=민첩성). 신제품이 계속 나오므로 평가를 빠르게 갱신해야 하는 비즈니스 동인 때문에 평가 부분이 별도 특성을 갖는다. 이 특성 클러스터를 가이드로 퀀텀(서비스 경계)을 분리한다.

```
  GG system:
   [ Public UI ]    -> scalability, availability, agility
   [ Back office ]  -> security, data integrity, auditability
   [ Assessment ]   -> maintainability, deployability, testability
   (each cluster = separate architecture quantum)
```

---

## 4. Scoping and the Cloud (범위 설정과 클라우드)

> 클라우드 자원은 많은 운영 특성을 캡슐화하므로 두 가지 시나리오를 고려해야 한다.

(1) 컨테이너 호스팅: 컨테이너의 특성과 오케스트레이션 도구(예: Kubernetes) 제약을 고려. (2) 클라우드 제공자 구성요소 조립: triggered functions·데이터베이스 등 제공자가 광고/유지하는 역량을 검토. 이전 세대가 물리 시스템에서 힘들게 얻은 탄력성 같은 역량이 이제 설정값이 됐지만, 제공자 가용성·보안이라는 새 트레이드오프가 생겼다.

---

## Summary (핵심 정리)

- 현대 아키텍처는 시스템이 아닌 퀀텀 단위로 특성 범위를 정의하며, 퀀텀은 독립 배포·고응집·저결합 단위다.
- 정적 결합은 퀀텀 경계를 정하고, 동기 통신은 특성이 다른 퀀텀 간에 타이밍 문제를 일으킨다.
- 특성 클러스터를 가이드로 퀀텀(서비스 경계)을 분리하면 스타일·입도·클라우드 선택의 트레이드오프를 잘 잡을 수 있다.
