# 11. The Modular Monolith Architecture Style

## 챕터 개요 (3줄 요약)

- 모듈러 모놀리스는 단일 배포 단위를 도메인별 모듈로 분할하는 도메인 분할(domain-partitioned) 아키텍처다.
- DDD(Domain-Driven Design) 확산으로 인기를 얻었으며, 단순성·저비용·모듈성이 강점이다.
- 모듈 구조·통신 방식과 거버넌스로 모듈 경계를 유지해 Big Ball of Mud로의 퇴화를 막는다.

---

## 1. Topology (토폴로지)

> 단일 배포 단위(WAR, EAR, .NET assembly)이지만 기능을 도메인 영역별로 그룹화한다.

계층형이 com.app.presentation.customer.profile처럼 기술 관심사로 조직한다면, 모듈러 모놀리스는 com.app.customer.profile처럼 도메인으로 조직한다(필요시 .presentation을 도메인 뒤에 둠).

```
   Single Deployment Unit
   +------------------------------------+
   | [OrderPlacement] [Inventory]       |
   | [Payment]        [Notification]    |
   | [Fulfillment]    [Shipping]        |
   +------------------------------------+
     (grouped by domain, not by tech layer)
```

---

## 2. Style Specifics (스타일 세부)

> 도메인(서브도메인)을 '모듈'이라 부르며, 모놀리식 구조와 모듈러 구조 두 가지로 조직할 수 있다.

### Monolithic Structure (모놀리식 구조)

모든 모듈이 단일 소스 저장소에 있고 한 단위로 배포. 가장 단순하나 코드 재사용·모듈 간 통신 과다로 경계가 무너져 Big Ball of Mud가 될 수 있어 엄격한 거버넌스 필요.

### Modular Structure (모듈러 구조)

각 모듈이 자체 산출물(JAR/DLL)이며 배포 시 결합. 팀별 분리 작업과 깨끗한 경계에 유리하나, 의존적 모듈 간 통신이 필요하면 효과가 떨어진다.

### Module Communication (모듈 통신)

- Peer-to-peer: 한 모듈이 다른 모듈 클래스를 직접 인스턴스화. 편리하지만 Big Ball of Mud / JAR(DLL) Hell 위험.
- Mediator(중재자): 추상화 계층으로 모듈을 디커플링. 단 각 모듈이 mediator에 결합됨(결합 단순화·재분배).

```
  Peer-to-peer:  Mod A ----> Mod B
  Mediator:      Mod A --> [Mediator] --> Mod B
```

---

## 3. Data / Cloud / Risks / Governance / Teams

> 보통 모놀리식 DB에 의존하나, 독립 모듈은 자체 DB를 가질 수도 있다.

클라우드: 모놀리식 특성상 온디맨드 프로비저닝 활용이 어려워 잘 맞지 않음(작은 시스템은 클라우드 서비스 일부 활용 가능). 위험: 너무 커지는 것(변경 지연, 예기치 못한 파손, 느린 시작), 과도한 코드 재사용(unstructured monolith), 과도한 모듈 간 통신(도메인 정의 부실 신호). 거버넌스: ArchUnit/NetArchTest/PyTestArch/TSArch로 (1) 네임스페이스가 정의된 모듈에 속하는지, (2) 모듈 간 의존 수 제한, (3) 특정 모듈 간 통신 금지를 자동 검증. 팀: 도메인 정렬 크로스펑셔널 팀에 최적(기술별 팀엔 부적합).

---

## 4. Style Characteristics (특성 평가)

> 비용·단순성·모듈성이 강점이나 모놀리식이라 확장성·탄력성은 낮다(quantum=1).

배포성·테스트성은 모듈성 덕에 계층형보다 약간 높은 2점. 탄력성·확장성은 1점. 내결함성 없음(일부 OOM이면 전체 크래시), 높은 MTTR.

```
  Cost/Simplicity/Modularity ***** (high)
  Deployability/Testability  **
  Scalability/Elasticity     *
```

---

## 5. When to Use / Not (사용 시점)

> 빠듯한 예산·시간, 새 시스템 시작, 도메인 중심 팀과 DDD에 적합하다.

방향이 불확실하면 모듈러 모놀리스로 시작해 추후 service-based/microservices로 전환하는 편이 낫다. 변경이 주로 도메인 기반일 때(예: 위시리스트 만료일 추가) 좋다. 부적합: 높은 확장성·탄력성·가용성·성능이 필요하거나, 변경이 주로 기술 지향(UI·DB 교체)일 때(이 경우 계층형이 나음). 예시: EasyMeals 음식 배달 시스템(PlaceOrder, Payment, PrepareOrder, Delivery, Recipes, Inventory 모듈 — 각 모듈은 1~다수 컴포넌트로 구성).

---

## Summary (핵심 정리)

- 모듈러 모놀리스는 단일 배포를 도메인별 모듈로 나누는 도메인 분할 스타일로 DDD에 적합하다.
- 모놀리식/모듈러 구조와 peer-to-peer/mediator 통신을 선택하며 거버넌스로 경계를 지킨다.
- 단순·저비용·모듈성이 강점이나 확장성·탄력성은 약해, 새 시스템 시작과 도메인 중심 팀에 좋다.
