# 05. Automated Testing

## 챕터 개요 (3줄 요약)
- 자동화 테스트는 문서 역할을 하고 유지보수성을 높이며 변경에 대한 자신감을 주는 코드베이스 투자다.
- 테스트는 단위(unit)→통합(integration)→E2E(end-to-end)의 피라미드 구조로 구성되며 위로 갈수록 느리고 깨지기 쉽다.
- 코드 커버리지는 맹목적 목표가 아닌 피드백 수단이며, 의미 있는 동작 검증 테스트 작성이 핵심이다.

---

## 1. Benefits of Automated Testing
> 자동화 테스트는 단순한 유행이 아니라 코드 품질을 높이고 엔지니어를 더 효율적으로 만드는 중요한 투자다.

- 추가 코드를 작성하는 것에 회의적일 수 있지만, 자동화 테스트는 코드 품질·자신감·효율성을 높이는 가치 있는 기술이다.

### Acts as Documentation
- 잘 작성된 테스트는 문서가 희박한 프로젝트에서 구원이 된다.
- shouldProcessCreditCardTransaction 같은 서술적 테스트 이름이 모듈 기능과 관련 코드 위치를 알려준다.
- 실패하는 테스트는 코드베이스 내 문제의 원인으로 안내한다.

### Improves Maintainability
- 테스트를 먼저 작성하는 것은 말하기 전에 생각하는 것과 같아 코드를 계획·구조화하게 한다.
- 테스트를 작성하려다 보면 단일 책임 원칙(single responsibility principle) 위반을 자연스럽게 발견한다.
- 테스트는 기능 검증뿐 아니라 더 나은 코드 설계로 안내한다.

### Boosts Your Confidence
- 견고한 테스트 스위트는 안전망(safety net)이 되어 변경·실험·리팩터링을 자신 있게 하게 한다.
- 테스트 없는 작업은 로프 없이 산을 오르는 것과 같다.
- 버그를 일찍(저렴할 때) 잡고 회귀(regression) 발생 가능성을 줄인다.

### Leads to Consistency and Repeatability
- 수동 테스트는 오류가 잦고 반복 불가능하며 노동 집약적이다.
- 자동화 테스트는 매번 동일한 단계를 정확히 실행하여 효과적인 회귀 테스트를 제공한다.

---

## 2. Types of Automated Testing
> 자동화 테스트는 단위, 통합, E2E 테스트의 피라미드(testing pyramid)로 구성되며 각 유형은 트레이드오프가 있다.

- Mike Cohn이 만든 테스팅 피라미드는 보유해야 할 테스트 유형의 시각적 가이드를 제공한다.
- 위로 갈수록 테스트는 느리고, 유지비가 크며, 깨지기 쉽다(단위는 마이크로초, E2E는 수 분).

```
        /\
       /E2E\        <- few, slow, expensive, brittle
      /------\
     /Integr. \     <- medium amount, broader scope
    /----------\
   / Unit Tests \   <- many, fast, isolated, cheap
  /--------------\
```

### Unit Tests
- 피라미드의 기반이며 가장 큰 비중을 차지하고, 컴포넌트를 격리하여 검증하는 계약(contract)과 같다.
- 빠르게 작성·실행되며 실패 시 특정 함수/라인을 가리켜 디버깅이 쉽다.

### Integration Tests
- 여러 컴포넌트/모듈이 하나의 단위로 함께 작동하는 방식을 검증한다.
- 개별 테스트된 컴포넌트가 결합될 때만 나타나는 문제를 잡아내며, 단위 테스트보다 느리고 덜 자주 실행된다.

### End-to-End Tests
- UI부터 백엔드까지 전체 애플리케이션을 다루며 실제 사용자 시나리오를 시뮬레이션한다.
- 느리고 자원 소모가 크며 깨지기 쉽지만(false negative), 시스템 전체 기능에 높은 확신을 준다.

### What You Should Not Test
- 언어 기능이나 프레임워크 코드는 테스트하지 않는다(유지관리자의 책임).
- getter/setter, 빌더, 자동 생성 DTO(Data Transfer Object) 같은 생성된 코드는 피한다.
- private 메서드를 직접 테스트하지 말고 이를 호출하는 public 인터페이스에 집중한다.
- 외부 서비스 의존 테스트는 피하고 mock 또는 test double을 사용한다.

---

## 3. Code Coverage
> 코드 커버리지는 테스트 실행 시 실행되는 코드 비율을 측정하는 메트릭이지만, 높은 수치가 곧 좋은 테스트를 의미하지는 않는다.

- 코드 커버리지는 비디오 게임의 지도처럼, 테스트가 실행한 코드 영역을 "밝혀" 사각지대를 드러낸다.
- IDE와 CI/CD(Continuous Integration/Continuous Delivery) 파이프라인에서 최소 임계값을 강제할 수 있다.
- line coverage, branch coverage 등으로 실행된 라인/분기/함수를 보고한다.
- 80~90% 같은 임계값 요구는 의미 없는 동작을 실행만 하는 "게이밍" 같은 역효과를 낳을 수 있다.
- 100% 커버리지(허영 메트릭)를 목표로 하지 말고, 주의가 필요한 영역을 안내하는 피드백 수단으로 사용한다.

---

## 4. Writing Tests
> 어떤 언어든 테스트 도구가 있으며(예: Java의 JUnit), 단언(assertion)으로 예상 결과를 검증한다.

- Java의 가장 인기 있는 프레임워크는 JUnit이며 Maven POM에 의존성을 선언해 사용한다.
- 테스트 우선(TDD, Test-Driven Development)은 "red-green-refactor" 사이클을 따른다: 실패 테스트→최소 구현→리팩터링.
- 테스트 나중(test-last)은 기능 구현 후 테스트를 작성하며, 두 방식 모두 단언으로 결과를 검증한다.
- AI 도구는 기존 스타일을 학습시켜 테스트 케이스 생성, 엣지 케이스 제안, 보일러플레이트 작성에 유용하다.

### Assertions & Unit Tests
- 단언(assertion)은 프로그램 실행 중 조건 충족을 검증하는 메서드다(예: assertEquals(expected, actual)).
- 테스트 대상은 SUT(System Under Test)라 부르며, 단위 테스트는 해당 클래스만 격리해 테스트한다.
- 엣지 케이스를 반드시 커버하여 사용자가 대신 발견하지 않게 한다.

### Mocking
- 모킹(mocking)은 의존성의 시뮬레이션 버전을 만들어 코드를 격리 테스트하는 기법이다.
- 의존성은 SUT가 작업 완료를 위해 의존하는 다른 클래스이며, Java에서는 Mockito가 인기 있다.
- @Mock으로 의존성을 모킹하고 @InjectMocks로 테스트 대상에 주입한다.

### Integration & E2E Tests
- 통합 테스트는 실제(non-mocked) 의존성으로 클래스 간 상호작용을 검증한다(이름에 Int 표기).
- E2E 테스트는 실제 환경(Spring Boot + 임베디드 Tomcat)에서 API 요청부터 DB까지 전체 흐름을 검증한다.

---

## Summary (핵심 정리)
- 자동화 테스트는 문서이자 안전망이며, 단위·통합·E2E 테스트가 각각 소프트웨어 품질 보장에 핵심 역할을 한다.
- 코드 커버리지는 임의의 숫자가 아닌 주의 영역 안내용 피드백으로 사용하고, 의미 있는 동작 검증에 집중한다.
- 건강한 생활 습관처럼 테스트를 습관으로 만들면 코드 품질, 유지보수성, 전문가로서의 성장에 측정 불가능한 이득을 준다.