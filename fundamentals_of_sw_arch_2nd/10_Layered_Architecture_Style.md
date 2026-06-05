# 10. Layered Architecture Style

## 챕터 개요 (3줄 요약)

- 계층형(layered, n-tiered) 아키텍처는 단순성·친숙성·저비용으로 레거시의 사실상 표준이다.
- 컴포넌트를 Presentation·Business·Persistence·Database의 수평 계층으로 기술적 분할한다.
- 단순하고 저렴하지만 규모가 커지면 확장성·배포성·테스트성이 급격히 나빠진다.

---

## 1. Topology (토폴로지)

> 컴포넌트를 역할별 수평 계층으로 조직하며, 보통 4개 표준 계층(Presentation, Business, Persistence, Database)으로 구성된다.

물리 배포는 세 가지 변형이 있다: (1) Presentation+Business+Persistence를 한 단위로, DB는 외부 분리. (2) Presentation을 분리하고 Business+Persistence를 한 단위로. (3) DB 포함 네 계층을 한 단위로(모바일·온프레미스 제품). 각 계층은 관심사 분리(separation of concerns)로 역할이 명확하나, 전체적 민첩성(agility)은 떨어진다. 기술적 분할이라 'customer' 같은 도메인이 모든 계층에 흩어져 DDD와 잘 안 맞는다.

```
   [ Presentation ]   <- UI / browser logic
   [ Business     ]   <- business rules
   [ Persistence  ]   <- data access (SQL)
   [ Database     ]   <- data store
```

명확한 구조 없이 "그냥 코딩 시작"하면 Architecture by Implication / Accidental Architecture 안티패턴에 빠지기 쉽다.

---

## 2. Style Specifics (스타일 세부)

> 계층은 닫힘(closed) 또는 열림(open)이며, 이는 격리 계층(layers of isolation) 개념과 연결된다.

### Layers of Isolation (격리 계층)

닫힌 계층은 요청이 계층을 건너뛸 수 없게 한다. 한 계층 변경이 다른 계층에 영향을 주지 않으려면(계약이 유지되는 한) 주요 흐름의 계층은 닫혀 있어야 한다. Presentation이 Persistence에 직접 접근하면 강결합·취약 아키텍처가 된다. 격리 계층 덕에 한 계층(예: UI 프레임워크)을 다른 계층 영향 없이 교체할 수 있다.

### Adding Layers (계층 추가)

공유 비즈니스 객체를 Presentation이 못 쓰게 하려면 Services 계층을 추가하되 '열림'으로 표시해, Business가 Services를 거치거나 건너뛰어 Persistence로 갈 수 있게 한다.

### Architecture Sinkhole Antipattern (싱크홀 안티패턴)

요청이 비즈니스 로직 없이 계층을 통과만 하는 경우. 80-20 규칙으로 판단: 싱크홀이 20%면 OK, 80%면 계층형이 부적합하다는 신호.

---

## 3. Data / Cloud / Risks / Governance / Teams

> 전통적으로 단일 모놀리식 DB와 결합하며, 모놀리식 특성상 여러 제약을 가진다.

데이터: 단일 모놀리식 DB. 클라우드: 계층 단위 배포 가능하나 온프레미스-클라우드 간 지연이 문제. 공통 위험: 내결함성 없음(일부 OOM이면 전체 크래시), 높은 MTTR(2~15분)로 가용성 저하. 거버넌스: ArchUnit의 layeredArchitecture()로 계층 접근 규칙을 자동 검증(이 스타일을 염두에 두고 만들어짐). 팀 토폴로지: 어떤 구성과도 잘 작동(stream-aligned가 전체 흐름 소유, enabling이 특정 계층 실험, complicated-subsystem이 Persistence로 분석 데이터 접근, platform은 모놀리스 비대화가 도전).

---

## 4. Style Characteristics (특성 평가)

> 비용과 단순성이 최대 강점이나, 규모가 커지면 평가가 빠르게 하락한다.

비용·단순성이 높다(모놀리식이라 분산보다 단순). 배포성·테스트성은 낮다(3줄 변경도 전체 재배포). 탄력성·확장성은 1점(quantum이 항상 1). 응답성은 캐싱·멀티스레딩으로 개선 가능하나 병렬 처리 부재·싱크홀로 3점.

```
  Cost/Simplicity  ***** (high)
  Deployability    **    (low)
  Testability      **    (low)
  Scalability      *     (very low, quantum=1)
  Elasticity       *     (very low)
```

---

## 5. When to Use / Not (사용 시점)

> 작고 단순한 애플리케이션, 빠듯한 예산·시간, 실현가능성(feasibility) 우선 상황에 적합하다.

작은 앱·웹사이트, 더 복잡한 스타일 검토 중 개발을 시작해야 할 때 좋다. 코드 재사용·상속 깊이를 낮게 유지해 추후 다른 스타일로 전환을 쉽게 한다. 대규모 시스템은 더 모듈러한 스타일이 낫다. 예시: 운영체제(Hardware/Kernel/System Call/User), OSI/TCP-IP 네트워크 계층.

---

## Summary (핵심 정리)

- 계층형은 기술적 분할로 Presentation·Business·Persistence·Database를 수평 분리하는 저비용·단순 스타일이다.
- 격리 계층(closed/open)과 싱크홀 안티패턴(80-20 규칙)이 핵심 설계 개념이다.
- 작고 단순한 시스템에 적합하나 확장성·배포성·테스트성이 약해 규모 확대 시 한계가 있다.
