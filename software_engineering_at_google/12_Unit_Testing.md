# 12. Unit Testing

## 챕터 개요 (3줄 요약)
- 단위 테스트는 좁은 범위(클래스·메서드)를 검증하며 빠르고 작아서, Google 테스트의 대부분(약 80%)을 차지하고 생산성을 극대화한다.
- 핵심 가치는 유지보수성(maintainability) — 깨지기 쉽지 않고(non-brittle) 명확한(clear) 테스트는 "그냥 작동"하며 실패 시 실제 버그를 가리킨다.
- AI 시대에 코드 변경이 잦을수록, 공개 API·상태 검증·행동 중심의 변하지 않는 테스트를 설계하는 것이 시니어의 핵심 레버리지다.

---

## 1. The Importance of Maintainability (유지보수성)
> 나쁜 테스트는 생산성을 깎는다 — 깨지기 쉬움(brittleness)과 불명확함(unclearness)이 두 가지 주요 문제다.

- Mary 시나리오: 무해한 변경이 수십 개 테스트를 깨뜨려 하루를 낭비 — 이는 작성자 잘못이 아니라 나쁜 테스트의 문제.
- 깨지는 테스트는 체크인 전에 고쳐야 하며, 그렇지 않으면 미래 엔지니어에게 짐이 된다.

---

## 2. Preventing Brittle Tests (깨지기 쉬운 테스트 방지)
> 이상적 테스트는 "변하지 않음(unchanging)" — 시스템 요구사항이 바뀔 때만 변경되어야 한다.

- 변경 4종류와 테스트 영향: 순수 리팩토링(테스트 불변), 새 기능(기존 테스트 불변), 버그 수정(기존 테스트 불변), 행동 변경(이때만 테스트 변경).
- Test via Public APIs: 구현 세부가 아니라 사용자처럼 공개 API를 호출 — 깨지면 실제 사용자도 깨진다는 명시적 계약 형성.
  - "public API"의 정의는 예술에 가까움: 헬퍼 클래스는 사용처를 통해 테스트, 일반 지원 라이브러리는 직접 테스트.
- Test State, Not Interactions: 결과 상태를 검증(state testing)하라 — 어떻게 도달했는지(interaction testing)는 깨지기 쉽다; mocking 프레임워크 남용 경계, 가능하면 실제 객체 사용.

```
State vs Interaction testing
Interaction (brittle):  verify(database).put("foobar");  // HOW
State (robust):         assertThat(accounts.getUser("foobar")).isNotNull(); // WHAT
```

---

## 3. Writing Clear Tests (명확한 테스트)
> 명확한 테스트는 존재 이유와 실패 이유가 즉시 드러난다 — 테스트는 작성자보다 오래 살아남는다.

- Complete and Concise: 본문에 이해에 필요한 모든 정보를 담되, 산만한 정보는 빼라(헬퍼로 무관 세부 숨김).
- Test Behaviors, Not Methods: 메서드당 테스트가 아니라 행동당 테스트 — "given/when/then"으로 표현; 메서드:행동은 다대다.
- 구조로 행동 강조: given(설정)/when(행동)/then(검증) 블록을 명시; 한 테스트는 보통 하나의 when/then.
- Name tests after behavior: 테스트 이름이 실패 보고서의 첫 토큰 — "shouldNotAllowWithdrawalsWhenBalanceIsEmpty"처럼 행동+결과를 서술("and"가 필요하면 테스트를 쪼개라).
- Don't Put Logic in Tests: 연산자·루프·조건문 같은 로직은 버그를 숨김(URL 이중 슬래시 예시) — 직선형 코드 + 약간의 중복 허용.
- Write Clear Failure Messages: 기대 vs 실제 상태와 맥락을 명확히(Truth 같은 어서션 라이브러리 활용).

---

## 4. DAMP, Not DRY (코드 공유)
> 테스트 코드는 DRY(반복 금지)보다 DAMP(Descriptive And Meaningful Phrases) — 명확성을 높이는 약간의 중복은 허용된다.

- 프로덕션 코드는 테스트 스위트가 보호하지만, 테스트는 스스로 서야 하므로 복잡성 비용이 더 크다.
- 너무 DRY한 테스트는 헬퍼에 세부를 숨겨 불완전·로직 은닉(버그 은폐) — DAMP로 재작성하면 각 테스트가 본문만으로 이해됨.
- 코드 공유 패턴:
  - Shared Values: 모호한 공유 상수 대신 기본값 헬퍼 메서드(named params / Builder 패턴)로 필요한 값만 명시.
  - Shared Setup: setUp으로 객체 구성은 좋지만, 특정 값에 의존하는 테스트는 그 값을 본문에서 직접 override.
  - Shared Helpers/Validation: 범용 validate() 남용은 행동 중심성 저하 — 단일 개념 사실을 검증하는 집중된 헬퍼만.
- Test Infrastructure: 여러 스위트가 의존하는 테스트 인프라는 프로덕션 코드처럼 다루고 자체 테스트 필요 — 조직 표준화(예: Mockito 단일화) 권장.

---

## Summary (핵심 정리)
- 변하지 않는 테스트를 지향하고, 공개 API로 테스트하며, 상호작용이 아닌 상태를 검증하라.
- 완전하고 간결하게, 메서드가 아닌 행동을 테스트하고 행동을 강조하는 구조로 작성하며, 행동을 따라 이름 짓고 로직을 넣지 마라.
- 명확한 실패 메시지를 쓰고 테스트 코드 공유는 DRY보다 DAMP를 따르라 — AI 시대 잦은 변경 속 코드 신뢰의 시니어 레버리지.
