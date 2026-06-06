# 13. Test Doubles

## 챕터 개요 (3줄 요약)
- 테스트 더블(test double)은 실제 구현을 대신하는 객체/함수로, 느리거나 flaky한 의존성 없이 빠른 small test를 가능하게 한다.
- 세 가지 트레이드오프(테스트 가능성·적용성·충실도)와 세 가지 기법(Faking·Stubbing·Interaction testing)이 핵심이며, 가능하면 실제 구현 > Fake > Stub/Interaction 순으로 선호한다.
- AI 시대에 테스트를 대량으로 빠르게 돌릴수록, 현실성(realism)과 격리(isolation)의 균형을 잡아 깨지지 않는 테스트를 설계하는 시니어 역량이 중요하다.

---

## 1. Impact & Concepts (영향과 기본 개념)
> 테스트 더블 사용에는 세 가지 트레이드오프가 따른다.

- Testability(테스트 가능성): 코드가 실제 구현을 더블로 교체할 수 있게 설계되어야 함 — Seam(이음새)을 통해 가능.
- Applicability(적용성): 오용하면 깨지기 쉽고 복잡하며 효과 없는 테스트 — 많은 경우 실제 구현이 낫다.
- Fidelity(충실도): 더블의 행동이 실제 구현과 얼마나 가까운가 — 완벽할 필요는 없으나 API 계약은 지켜야 함.
- Seam & Dependency Injection: 의존성을 직접 생성하지 않고 주입(Guice/Dagger)해 테스트에서 더블로 교체.
- Mocking Framework(Mockito/googlemock/unittest.mock): 더블 생성 보일러플레이트를 줄이지만, 남용하면 유지보수가 어려워짐.

---

## 2. Three Techniques (세 가지 기법)
> 적절한 기법을 아는 엔지니어가 상황에 맞게 더블을 선택할 수 있다.

```
Test Double Techniques
Faking      : lightweight working impl (e.g. in-memory DB) - highest fidelity
Stubbing    : hardcode return values via when(...).thenReturn(...)
Interaction : verify a function was CALLED correctly (verify(...))
```

---

## 3. Real Implementations (실제 구현 우선)
> 첫 선택은 실제 구현 — 프로덕션처럼 코드를 실행해 충실도(fidelity)가 높다(classical testing).

- "Prefer Realism Over Isolation" — 더블 과의존은 통합테스트/수동검증을 강요해 개발을 늦추거나 버그를 흘려보냄.
- 좋은 테스트는 구현 구조가 아니라 API 관점에서 작성 — 실제 구현의 버그로 테스트가 실패하는 것은 바람직하다.
- @DoNotMock: API 소유자가 "이 타입은 mock하지 말라"고 선언 — 수만 번 mock되면 API 변경이 불가능해지는 것을 방지.
- 실제 구현 선택 기준: 빠르고(execution time), 결정적(determinism, hermetic·시계 비의존)이며, 의존성 구성(dependency construction)이 단순할 때 — value object는 항상 실제 구현.

---

## 4. Faking (페이크)
> 실제 구현이 불가능하면 Fake가 최선 — 시스템이 실제인지 페이크인지 구분 못 할 만큼 유사하게 동작한다.

- Fake는 실제 구현 소유 팀이 작성·유지(behavior 동기화 필요); 보통 테스트 불가능 코드의 "뿌리(root)"에만 작성.
- Fidelity의 핵심: "테스트 관점에서" API 계약에 완벽 충실(입력→출력·상태변화 동일) — latency/리소스 등은 완벽할 필요 없음.
- Fake도 테스트되어야 함: contract test(같은 공개 인터페이스 테스트를 실제 구현과 Fake 양쪽에 실행).
- Fake가 없으면: 소유자에게 요청 → 직접 작성(API 호출을 한 클래스로 래핑) → 실제 구현/다른 기법으로 폴백.

---

## 5. Stubbing (스터빙)
> 행동 없는 함수에 반환값을 하드코딩 — 빠르고 쉽지만 남용 위험이 크다.

- 남용의 위험: 테스트가 불명확(구현 세부 노출)·깨지기 쉬움(brittle)·효과 저하(계약 충실도 보장 불가, 상태 저장 불가).
- 신호: 함수가 왜 stub됐는지 이해하려 시스템을 머릿속으로 따라가야 한다면 부적절.
- 적절한 경우: 시스템을 특정 상태로 만들기 위해 특정 반환값/에러가 필요할 때 — stub 함수는 어서션과 직접 관계, 소수만 사용.

---

## 6. Interaction Testing (상호작용 테스트)
> 함수가 "어떻게 호출됐는가"를 검증 — 가능하면 피하고 상태 테스트(state testing)를 선호한다.

- State testing이 우월: 시스템이 실제로 올바르게 동작하는지 검증(예: 저장 후 조회); interaction은 "호출됐다"만 검증.
- 단점: 구현 세부 노출로 "change-detector test"(행동 불변에도 코드 변경마다 실패)가 됨.
- 적절한 경우: 실제 구현/Fake 사용 불가, 또는 호출 횟수/순서가 동작에 영향(예: 캐싱이 DB 호출을 줄이는지 verify).
- 베스트프랙티스: 상태 변경(state-changing) 함수에만 interaction test(비상태변경 함수는 redundant), 과명세(overspecification) 회피(eq/any로 관련 인자만 검증).

---

## Summary (핵심 정리)
- 실제 구현을 테스트 더블보다 우선하고, 실제 구현이 불가능하면 Fake가 이상적 해법이다.
- Stubbing 남용은 불명확·깨지기 쉬운 테스트를 낳고, Interaction testing은 구현 세부를 노출하므로 가능하면 피하라.
- AI 시대에 테스트를 빠르고 대량으로 돌리되, 현실성과 격리의 균형으로 깨지지 않는 테스트를 설계하는 것이 시니어 레버리지다.
