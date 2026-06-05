# 01. Programmer to Engineer

## 챕터 개요 (3줄 요약)

- 단순히 코드를 작성하는 programmer/coder를 넘어, SDLC(Software Development Life Cycle) 전반에 engineering discipline을 적용하는 software engineer로 성장하는 길을 다룬다.
- 학습 경로(대학, boot camp, 독학)는 중요하지 않으며, 학교에서 가르치지 않는 협업·코드 읽기·레거시 코드 작업 같은 fundamentals이 장기적 성공을 좌우한다.
- 성급한 solutioning을 피하고, 근본 원인(why)을 파악하며, 다음 개발자를 배려하는 Golden Rule을 코드에 적용하는 실천적 조언을 제공한다.

---

## 1. An Engineer by Any Other Name

> programmer/coder, developer, software engineer는 동의어가 아니며, engineer는 SDLC 전체에 engineering discipline을 적용하는 사람이다.

- programmer 또는 coder는 주로 코드 생성이라는 단일 작업에 집중하며, 특정 언어나 framework에 능숙하지만 SDLC 전체를 이해하지 못할 수 있다.
- developer는 소프트웨어 전달의 큰 그림을 더 잘 이해하고, 여러 언어·framework와 다양한 비즈니스 도메인 경험을 갖춘 경우가 많다.
- software engineer는 scalability(확장성), reliability(신뢰성), efficiency(효율성), security(보안)를 고려하며 가장 복잡하고 중요한 시스템을 책임진다.
- engineer는 규칙뿐 아니라 규칙을 언제 구부리거나 깨야 하는지도 안다.
- boot camp와 대학은 주로 코드 작성의 기계적 측면에 집중하여 programmer/developer를 양성하며, 이는 경력의 시작점일 뿐이다.

## 2. Fundamentals Matter

> 프로 운동선수가 기본기에 집중하듯, 소프트웨어에도 마스터해야 할 핵심 원칙이 존재한다.

- 경력의 대부분은 기존 코드와 함께 일하므로, 남이 작성한 코드를 읽고 새 codebase를 빠르게 이해하는 능력이 필수적이다.
- 낯선 codebase는 위협적이지만 ambiguity(모호함)에 익숙해져야 하며, 변경 전에 모든 것을 이해할 필요는 없다.
- 다음 사람을 위해 단순화하는 코드를 작성하는 법은 잘 가르쳐지지 않는다.
- 많은 programmer가 최신 기술에 집착하면서 승진에 도움이 되는 soft skills을 무시한다.
- fundamentals은 화제성은 떨어지지만 경력 정체와 더 큰 기회의 차이를 만든다.

## 3. The Many Paths to Becoming a Software Engineer

> engineer가 되는 길은 여러 가지이며, 어떤 경로로 배웠는지는 중요하지 않다.

- 대학 학위, boot camp, 독학, 관련 분야(전기공학 등) 전공 후 전향 등 다양한 경로가 존재한다.
- 학부 CS(Computer Science) 프로그램은 대학원 진학을 염두에 두고 algorithm, 언어 설계, compiler theory, OS(Operating System)에 집중하지만 실무 준비는 부족하다.
- boot camp는 짧은 기간에 당대의 언어·framework를 집중적으로 다루어 더 실용적이나 지식의 수명이 짧을 수 있다.
- 시간 투자: boot camp 약 600시간, associate 학위 약 850시간, bachelor 약 1,400시간.
- 성공은 problem-solving, tinkering(만지작거리며 탐구), creativity에 달려 있으며 수학 능력보다 언어 능력이 더 중요하다.

## 4. What You Were Taught Versus What You Need to Know

> 학습 경로와 무관하게 코드 작성법은 배우지만, 협업·코드 읽기·레거시 작업 같은 중요한 것들은 빠져 있다.

- 대부분의 교육은 technical debt이 없는 greenfield 프로젝트에서 이루어진다.
- 실무에서는 대부분의 시간을 legacy 또는 heritage 애플리케이션과 함께 보내며, 과거 결정의 부담을 떠안는다.
- 실제 프로젝트는 수십만~수백만 줄의 코드로 전 세계 팀이 함께 작업한다.
- 코드는 채점용이 아니라 production에서 실제 사용자가 의존하며 비즈니스 가치를 전달한다.
- 코드는 대부분의 수명을 maintenance phase(유지보수 단계)에서 보내므로, 소프트웨어는 명확한 시작과 끝이 있는 project가 아니라 지속 투자되는 product에 가깝다.

## 5. Embrace the Lazy Programmer Ethos

> lazy programmer ethos는 게으름이 아니라 효율성에 집중하는 철학이다.

- 많은 신입 engineer가 문제 도메인을 고려하지 않고 즉시 코드 작성으로 돌진한다.
- 전략적으로 게으르게 굴면 생각할 시간이 생겨 더 나은 솔루션을 만든다.
- brute-force 접근을 경계하고 설계를 반복(iterate)하며, big O notation으로 best/worst/average case를 고려한다.
- 대개 같은 문제를 처음 푸는 사람이 아니므로, 기존 솔루션이나 library를 먼저 검색하라.
- 10분의 검색이 며칠의 작업을 아낄 수 있다(저자 Nate의 capitalization 함수 일화).

## 6. The Value of a Fresh Set of Eyes

> 신입 개발자는 역사적 편견에 얽매이지 않은 신선한 관점이라는 강점을 가진다.

- 새로운 사람은 더 이상 유용하지 않은 과거 결정에 구애받지 않고 문제를 본다.
- 이런 관점은 혁신적 솔루션, 비효율 발견, 오래된 가정에 대한 도전으로 이어진다.
- "We've always done it this way"는 조직에서 가장 위험한 말이다(Grace Hopper).
- 저자 Nate의 일화: 독립적인 widget을 병렬 처리하자는 단순한 제안으로 야간 batch 성능 문제를 해결했다.
- 이상하거나 맞지 않는 것을 보면 학습 의도로 질문하라.

## 7. Don't Solution Too Quickly

> 문제를 만나면 성급히 해결책으로 뛰어들지 말고 root cause(근본 원인)를 이해하는 데 시간을 들여라.

- quick fix는 즉각적 문제는 해결하지만 장기적으로 더 많은 문제를 만든다.
- "shifting left"는 가치 있는 활동(testing, review, security)을 프로세스 초반으로 이동시키는 것이다.
- production에서 발견된 버그는 가장 비싸며, 데이터 손상과 복잡한 refactoring을 유발한다.
- 소프트웨어의 큰 문제는 misconception(오해) 문제이며, 어떤 test나 type system도 이를 잡지 못한다.
- customer가 요청하는 것과 실제 비즈니스 문제를 해결하는 것 사이에는 종종 괴리가 있다.

### The Five Whys

- Toyota Production System에서 유래한 기법으로, "왜?"를 5번 반복하여 표면 증상 아래의 근본 문제를 발굴한다.

```
Problem  -->  Why? -->  Why? -->  Why? -->  Why? -->  Why?
(symptom)                                         (root cause)
   |                                                   |
   +--- dig beneath surface to find real problem ------+
```

## 8. Apply the Golden Rule to Software

> 남에게 대접받고 싶은 대로 대접하라는 Golden Rule을 코드에도 적용하라.

- 뒤를 이을 개발자(그것이 미래의 자신일 수도 있다)를 생각하라.
- 코드는 궁극적으로 communication mechanism이며, 컴파일러가 아닌 인간을 위해 최적화하라.
- clean code를 작성하고 문서를 최신으로 유지하며 명확한 diagram을 만들어라.
- 질문에 답하고 다른 사람을 돕는 데 시간을 내라.
- 이를 실천하면 주변을 더 낫게 만드는 사람으로 팀에서 수요가 높아진다.

---

## Summary (핵심 정리)

- software engineer가 된다는 것은 문법적으로 올바른 코드를 넘어 SDLC 전반의 균형 잡힌 역량을 갖추는 것이며, 학습 경로보다 그 역량을 어떻게 결합하느냐가 중요하다.
- 성급한 solutioning, overengineering, brute-force 같은 흔한 함정을 피하고, fundamentals과 신선한 관점을 무기로 삼아야 한다.
- mentor를 찾고, 코드 전에 문제를 이해하며, 추상화 아래 계층을 들여다보는 실천이 programmer에서 engineer로의 성장을 돕는다.
