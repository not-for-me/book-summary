# 18. Microservices Architecture

## 챕터 개요 (3줄 요약)

- 마이크로서비스는 DDD(Domain-Driven Design)의 경계 컨텍스트(bounded context)에서 영감받은 'share nothing' 분산 스타일이다.
- 단일 목적의 작은 서비스가 자체 데이터와 의존성을 포함해 독립 배포·실행되며, 재사용보다 중복을 선호한다.
- 확장성·탄력성·진화성·내결함성이 강점이나 성능·서비스 입도·트랜잭션이 난제다.

---

## 1. Topology / Bounded Context / Granularity

> 각 서비스가 기능·서브도메인·워크플로우를 모델링하며, 자체 DB까지 포함하는 경계 컨텍스트를 물리적으로 구현한다.

극단적 디커플링이 목표라 재사용 대신 중복(예: Address 클래스 복제)을 택한다. 입도(granularity)가 성공의 핵심 — 'micro'는 라벨이지 명령이 아니다(Martin Fowler). 너무 작으면 통신 링크가 늘어 Big Ball of Distributed Mud가 된다. 경계 가이드: Purpose(기능 응집), Transactions(트랜잭션 경계), Choreography(통신 과다 시 통합). 반복(iteration)이 유일한 좋은 설계 방법.

```
   [API Gateway]  (routing + cross-cutting only, NO business logic)
   [Svc A + DB-A] [Svc B + DB-B] [Svc C + DB-C]
     (bounded context: code + schema + own database)
```

---

## 2. Style Specifics (스타일 세부)

> 데이터 격리, API 계층, 운영 재사용, 통신, 코레오그래피/오케스트레이션, 사가가 핵심 요소다.

### Data Isolation (데이터 격리)

공유 스키마·DB를 통합 지점으로 쓰지 않음. Entity Trap 주의. 단일 진실 원천을 정하거나 복제·캐싱으로 분산. 장점: 팀별 최적 DB 기술 선택 자유.

### API Layer (API 계층)

소비자와 서비스 사이의 API Gateway(역프록시 또는 게이트웨이). 라우팅·횡단 관심사만, 비즈니스 로직·중재·오케스트레이션 금지(팁).

### Operational Reuse (운영 재사용)

도메인은 중복, 운영(모니터링·로깅·서킷브레이커)은 공유. Sidecar 패턴으로 공통 운영 관심사를 각 서비스에 주입 → service mesh(Istio 같은 service plane으로 연결). Service discovery로 탄력성 구현.

### Communication (통신)

동기/비동기 결정. protocol-aware(호출 프로토콜 인지), heterogeneous(폴리글랏 지원), interoperability(서비스 간 호출). 비동기는 이벤트·메시지 사용.

### Choreography vs Orchestration

- Choreography(코레오그래피): 중앙 조정자 없이 서비스가 직접 호출. 디커플링 유지하나 오류 처리·조정 복잡(복잡 시 Front Controller 패턴).
- Orchestration(오케스트레이션): 전용 중재자 서비스로 조정. 결합 발생하나 조정을 한 곳에 집중.

### Transactions and Sagas (트랜잭션과 사가)

서비스 경계 넘는 트랜잭션은 피하라 — 입도를 고치는 게 답(Connascence of Values 유발). 불가피하면 Saga 패턴: 중재자가 각 호출의 성공/실패를 조정하고, 실패 시 보상 트랜잭션(compensating transaction)으로 되돌림. 트랜잭션이 지배적이면 마이크로서비스가 부적합.

---

## 3. Data Topologies (데이터 토폴로지)

> 마이크로서비스는 데이터 분리를 '요구'하는 유일한 스타일로, Database-per-Service가 표준이다.

모놀리식 DB는 불가능(60개 서비스 공유 시 변경 제어 악몽, 경계 컨텍스트 파괴, 확장성·연결·가용성 문제). Database-per-Service는 경계 컨텍스트 보존, 변경 격리, DB 기술 자유 변경, 확장성·내결함성 우수. 불가피하면 5~6개 서비스까지 DB 공유 가능(더 넓은 경계 컨텍스트), 단 스키마 변경 조정 부담.

```
  Database-per-Service:
   [Svc A]->[DB-A]  [Svc B]->[DB-B]  [Svc C]->[DB-C]
   (each owns its data; others request via contract)
```

---

## 4. Cloud / Risks / Governance / Teams

> 'cloud-native' 아키텍처로 불릴 만큼 클라우드에 적합하다.

클라우드: 온디맨드 프로비저닝과 잘 맞음. Serverless(AWS Lambda 등)는 별도 스타일이 아니라 마이크로서비스의 배포 모델로 간주. 위험: 너무 작은 서비스(Grains of Sand 안티패턴), 과도한 서비스 간 통신, 과도한 데이터 공유, 코드 재사용(공유 라이브러리가 경계 컨텍스트를 깨뜨림). 거버넌스: 정적 결합(공유 라이브러리·계약)과 동적 결합(로그·레지스트리로 서비스 간 호출 추적) 모니터링. 팀: 도메인 정렬 크로스펑셔널 팀에 최적(platform/enabling 팀이 sidecar·service mesh 담당).

---

## 5. Style Characteristics (특성 평가)

> 확장성·탄력성·진화성·내결함성·배포성·테스트성이 강점, 성능이 약점이다.

DevOps 혁명 없이는 존재 불가(자동 배포·테스트). 단일 목적 서비스로 높은 내결함성. 확장성·탄력성·진화성 우수(극단적 디커플링이 빠른 변화 지원). 성능은 이슈(네트워크 호출·엔드포인트 보안·데이터 지연 — 그래서 캐싱·복제·코레오그래피 활용). 도메인 분할이며 현대 아키텍처 중 가장 뚜렷한 quanta를 가짐.

```
  Scalability/Elasticity/Evolvability *****
  Fault Tolerance/Deployability/Testability ****~*****
  Performance  low (network/security/data latency)
```

---

## 6. Examples and Use Cases (예시)

> 기능·데이터 모듈성이 높은 시스템에 적합하다.

환자 생체신호 모니터링 시스템: 각 생체신호(심박·혈압·산소)를 독립 마이크로서비스로, 자체 데이터 저장. 공통 Alert Staff·Display Vital Signs는 공유 서비스. 강점 입증: 내결함성(한 서비스 다운이 타 서비스 영향 없음), 테스트성(혈압 서비스 유지보수 범위 작음), 진화성(새 생체신호 추가 용이).

---

## Summary (핵심 정리)

- 마이크로서비스는 경계 컨텍스트 기반의 share-nothing 스타일로 재사용보다 중복을 택해 극단적 디커플링을 추구한다.
- Database-per-Service, Sidecar/service mesh, 코레오그래피/오케스트레이션, Saga 패턴이 핵심 요소다.
- 확장성·탄력성·진화성·내결함성이 강점이나 성능·입도·트랜잭션 관리가 주요 난제다.
