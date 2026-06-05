# 02. Reading Code

## 챕터 개요 (3줄 요약)

- software engineer는 코드를 쓰는 것보다 읽는 데 훨씬 많은 시간을 쓰며, 대부분 기존 코드(brownfield/legacy)와 함께 일한다.
- 낯선 코드를 마주할 때 IKEA effect, mere-exposure effect 같은 cognitive bias가 작동함을 인식하고, 문서·tests·software archeology로 체계적으로 접근해야 한다.
- IDE(Integrated Development Environment) 기능과 tests를 living documentation으로 활용하며, AI는 보조 도구일 뿐 개발자가 pilot로서 trust but verify 해야 한다.

---

## 1. The Challenge of Working with Existing Code

> 기존 코드 문제를 풀 때는 사실 네 가지 문제를 동시에 풀어야 한다.

- 실무 대부분은 blank editor가 아닌 brownfield development, 즉 legacy code와 함께하는 작업이다.
- 첫째, 해결하려는 business problem 자체를 이해해야 한다.
- 둘째, 이전 개발자의 시각으로 문제를 봐야 하며, 이것이 가장 어려운 부분이다(서로 다른 스타일·pattern·library).
- 셋째, 기존 코드의 abstraction 수준이 부적절하여 refactoring이 필요할 수 있다.
- 넷째, archaeologist처럼 technical debt와 anti-pattern 층을 벗겨내며 작성 당시 맥락으로 코드를 봐야 한다(코드의 "carbon date").
- legacy code는 비하적 표현이며, 저자들은 heritage code 또는 existing code라는 긍정적 표현을 선호한다.

## 2. Cognitive Biases

> 기존 코드를 읽을 때 자신의 코드를 과대평가하게 만드는 인지 편향이 작동한다.

- IKEA effect: 스스로 만든 것에 더 높은 가치를 매기며, 한 연구에서 직접 조립한 제품에 63% 더 지불했다.
- mere-exposure effect: 이미 익숙한 것을 선호하며, 이는 프로그래밍 언어에 대한 독단(dogmatism)으로 이어진다.
- Paul Graham의 "Beating the Averages" 에세이의 Blub paradox: Blub 프로그래머는 아래 언어는 기능 부족으로, 위 언어는 불필요한 기능으로 본다.
- 동료가 특정 도구를 고집할 때 IKEA effect나 Blub paradox인지 질문해보라.
- 자기 자신도 이런 편향을 보이지 않는지 성찰해야 한다.

## 3. Approaching Unfamiliar Code

> 새 프로젝트에 적응할 때는 팀원, 문서, ADR(Architecture Decision Records)부터 시작하라.

- onboarding의 일부로 기본 프로젝트 개요를 팀원에게 받아라.
- README, wiki, ADR은 결함·장애의 와중에 사라지는 중요한 "why"를 제공한다.
- 문서가 오래되었으면 배우면서 갱신하고, 없으면 직접 만들어 Golden Rule을 실천하라.
- 문서화할 질문: 무엇을 하는가, 어떻게 동작하는가, 무엇에 의존하는가, 어떻게 실행하는가.
- tests를 documentation으로 사용하면 코드와 함께 진화하는 executable documentation을 만들 수 있다.
- code coverage 같은 metric은 오도할 수 있다(assert 없는 92% coverage 일화) — metric의 본질적 가치를 잊지 마라.

## 4. Software Archeology

> 팀과 문서를 살핀 후 editor를 열고 codebase를 직접 파헤치는 software archeology를 실천하라.

- 코드 구조를 살펴라: monolith인가 분산 architecture인가, 어떤 domain 개념이 표현되는가.
- 의도가 불분명하면 누가 함수를 호출하는지 역추적하여 user action과 코드의 연결을 찾아라.
- 애플리케이션을 실행하고 UI 요소나 parameter를 코드로 mapping하며 mental model을 뇌에 로드하라.
- Spring PetClinic 예시: findOwners.html → /owners endpoint → OwnerController의 processFindForm에 breakpoint를 걸어 추적한다.
- 이름이 실제 동작을 반영한다고 가정하지 말고 hunch를 확인하라(매우 high-level exception은 문제를 가린다).
- source control 도구(git log, git blame)로 자주 수정되는 클래스와 commit log를 분석하라.

```
Survey team  -->  Read docs/ADR  -->  Open editor (archeology)
     |                                       |
     v                                       v
 Run the app  -->  Map UI/param to code  -->  Build mental model
```

## 5. Effective Code-Reading Strategies

> IDE 기능을 적극 활용하고 tests를 분석하여 코드를 이해하라.

- IDE는 단순 text editor가 아니며, 매주 새로운 기능을 하나씩 배우는 continuous learning을 권장한다(IntelliJ Tip of the Day).
- 핵심 navigation 기능: Find Usages/References, Jump to Definition, Call Hierarchy, Type Hierarchy, Dependency diagram.
- 자동 inspection은 refactoring을 제안하며 언어의 idiomatic pattern을 가르쳐준다(for 루프 → forEach).
- structure view로 클래스의 메서드를 한눈에 파악할 수 있다.
- tests as living documentation: CI(Continuous Integration)를 통과해야 하므로 항상 최신 상태인 신뢰할 수 있는 진실의 원천이다.

### Reading Tests for Insight

- tests는 코드가 "어떻게 동작해야 하는지"를 알려주며, 구현 전 맥락을 제공한다.
- integration tests는 component들이 비즈니스 요구를 충족하는 큰 그림(workflow)을 보여준다.
- shouldHandleEmptyList, shouldRejectInvalidInput 같은 이름의 test는 edge case와 시스템 한계를 드러낸다.

## 6. Practice Makes Perfect & AI

> 코드 읽기 실력은 연습으로만 향상되며, AI는 만능이 아니다.

- 거대한 codebase를 한 사람이 완전히 이해할 수는 없으며, 그것이 목표도 아니다.
- 향상의 지름길은 없고 open source 코드를 많이 읽는 것뿐이며, 시간이 지나면 쉬워진다.
- GitHub Copilot, ChatGPT 같은 도구는 코드 조각 설명에 유용하지만 enterprise codebase의 맥락은 모른다.
- AI는 hallucination 위험이 있어, 개발자가 pilot로서 trust but verify 해야 한다.
- 현대 애플리케이션은 여러 repository와 library에 걸쳐 있어 개발자가 관계와 복잡성을 이해해야 한다.

---

## Summary (핵심 정리)

- 코딩은 거꾸로 가르쳐져 읽기보다 쓰기를 먼저 배우지만, 경력의 상당 부분은 남이 쓴 코드를 읽는 데 쓰인다.
- cognitive bias를 경계하고, 문서·software archeology·IDE 기능·tests를 활용해 낯선 codebase를 체계적으로 탐색하라.
- 이해가 깊어지면 다음 개발자(아마도 미래의 자신)를 위해 코드를 발견했을 때보다 더 낫게 남겨라.
