# 14. Service-Based Architecture Style

## 챕터 개요 (3줄 요약)

- 서비스 기반 아키텍처는 마이크로서비스의 하이브리드 변형으로, 가장 실용적인 분산 스타일로 평가된다.
- 분리 배포된 UI, 굵은 입도(coarse-grained)의 도메인 서비스, 선택적 모놀리식 DB로 구성된다.
- 마이크로서비스만큼 복잡·고비용이 아니면서 분산의 이점을 누려 비즈니스 앱에 인기다.

---

## 1. Topology (토폴로지)

> 분산 매크로 계층 구조로, 분리 배포된 UI·원격 도메인 서비스·(선택적) 모놀리식 DB로 구성된다.

서비스는 도메인/서브도메인을 나타내는 '도메인 서비스'로 굵은 입도이며 독립·분리 배포된다. 컨테이너화 불필요(가능은 함). 단일 모놀리식 DB 사용 시 도메인 서비스를 12개 이하로 권장. UI는 REST 등 원격 프로토콜로 접근하며 service locator 패턴을 내장한다.

```
   [ User Interface ]
        |  (REST)
   [Domain Svc A][Domain Svc B][Domain Svc C]  (coarse-grained)
        \        |        /
         [ Monolithic Database ]
```

---

## 2. Style Specifics (스타일 세부)

> 도메인 서비스는 보통 계층형(API Facade/Business/Persistence) 또는 서브도메인 분할로 설계된다.

API 접근 파사드가 UI 요청을 내부적으로 오케스트레이션한다(예: OrderService가 주문·결제·재고를 클래스 수준에서 조율). 마이크로서비스가 외부 서비스 오케스트레이션인 것과 대비된다. 굵은 입도 덕에 단일 도메인 서비스 내에서 표준 ACID(Atomicity, Consistency, Isolation, Durability) 트랜잭션으로 무결성 보장(만료 카드 시 롤백). 마이크로서비스는 BASE(Basic availability, Soft state, Eventual consistency) 트랜잭션과 보상 업데이트가 필요. 트레이드오프: 굵은 서비스 변경은 전체 서비스 재테스트·재배포 필요. UI 변형(단일~도메인별 분리)과 API Gateway(횡단 관심사 통합, 로드밸런싱) 옵션이 유연성을 더한다.

---

## 3. Data / Cloud / Risks / Governance / Teams

> 분산이면서 모놀리식 DB를 효과적으로 지원하는 유일한 스타일이며, 데이터 토폴로지가 유연하다.

데이터: 단일 DB ~ 도메인별 DB. 엔티티 객체(스키마)를 단일 공유 라이브러리에 두는 것은 안티패턴(변경이 모든 서비스에 파급). 대신 DB를 논리적으로 분할해(common/customer/invoicing/order/tracking) 도메인별 공유 라이브러리로 매핑하고, 가능한 한 세분화하라(팁). 공통 엔티티는 DB 팀만 변경하도록 버전 관리에서 잠근다. 클라우드: 분산이라 적합(컨테이너 서비스로 구현). 위험: 서비스 간 통신 회피(과다하면 도메인 분할 오류 신호), 도메인 서비스 12개 초과. 거버넌스: 변경이 여러 도메인 서비스에 걸치지 않도록, 서비스 간 통신량 제어. 팀: 도메인 정렬 크로스펑셔널 팀에 최적.

---

## 4. Style Characteristics (특성 평가)

> 5점은 없지만 민첩성·테스트성·배포성·내결함성·가용성이 4점으로 높고, 단순성·저비용이 차별점이다.

도메인 분할. 분산이라 quanta >= 1(공유 DB/UI면 1 quantum, 페더레이션하면 다중). 민첩성·테스트성·배포성 4점(도메인 스코핑 모듈성). 내결함성·가용성 4점(서비스 자기완결, 서비스 간 통신 적음 — 한 서비스 다운이 타 서비스에 영향 없음). 확장성 3점, 탄력성 2점(굵은 입도라 기능 복제 많음, 보통 단일 인스턴스). 단순성·비용이 마이크로서비스 등보다 우수해 가장 구현하기 쉬운 분산 아키텍처. ACID를 가장 잘 활용하는 분산 스타일이며 DDD에 자연스럽게 맞는다.

```
  Agility/Testability/Deployability  ****
  Fault Tolerance/Availability       ****
  Simplicity/Cost                    **** (vs other distributed)
  Scalability ***  Elasticity **
```

---

## 5. Examples and Use Cases (예시)

> Going Green 전자기기 재활용 시스템으로 유연성을 보여준다.

도메인별 서비스(Quoting, Receiving, Assessment, Accounting, ItemStatus, Recycling, Reporting). 고객 대면 Quoting/ItemStatus만 다중 인스턴스로 확장, 나머지는 단일. UI를 Customer Facing/Receiving/Recycling-Accounting으로 분리하고 외부·내부 DB를 분리(네트워크 존 분리, 보안, 별도 quantum). Assessment는 자주 변하므로 격리되어 민첩성 확보. 마이크로서비스로 가기 전 '디딤돌(stepping stone)'로도 유용하다("모든 부분이 마이크로서비스일 필요는 없다" - Mark).

---

## Summary (핵심 정리)

- 서비스 기반 아키텍처는 굵은 입도의 도메인 서비스로 구성된 실용적·저비용 분산 스타일이다.
- ACID 트랜잭션과 모놀리식 DB를 활용 가능하며 DDD·도메인 정렬 팀에 잘 맞는다.
- 민첩성·내결함성·단순성이 강점이고, 마이크로서비스로의 디딤돌로도 좋다.
