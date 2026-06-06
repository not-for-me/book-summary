# 11. Testing Overview

## 챕터 개요 (3줄 요약)
- 개발자 주도 자동화 테스트는 버그를 일찍 잡을 뿐 아니라, "변화를 가능하게(support change)" 만들어 빠른 반복과 자신감 있는 리팩토링을 뒷받침한다.
- 테스트는 크기(Size: small/medium/large)와 범위(Scope: unit/integration/e2e)라는 두 축으로 분류되며, 작고 좁은 테스트를 선호하는 테스트 피라미드(80/15/5)가 이상적이다.
- AI 시대에 코드 생산·변경 속도가 폭증할수록, 자동화 테스트로 안전망을 설계하는 것이 빠른 적응과 제품 신뢰의 핵심 레버리지가 된다.

---

## 1. Why Do We Write Tests? (왜 테스트하는가)
> 테스트의 가치는 엔지니어의 신뢰에서 나오며, 나쁜 테스트 스위트는 없느니만 못하다.

- 단순 테스트 4요소: 단일 행동, 특정 입력, 관찰 가능한 출력, 통제된 환경.
- GWS(Google Web Server) 사례: 2005년 엔지니어 주도 자동 테스트 정책 도입 후 1년 만에 긴급 푸시 절반으로 감소(이전 80%+ 푸시에 버그).
- 핵심 통찰: 개인 능력만으로 결함을 막을 수 없다 — 100명이 월 1버그만 내도 매일 5개 새 버그; 테스트는 집단 지혜를 자산화한다.
- 현대 개발 속도(하루 수차례 배포, 수많은 플랫폼/언어/기기)에서는 자동화가 유일한 답.

---

## 2. Write, Run, React & Benefits (작성·실행·반응과 이점)
> 자동 테스트는 작성→실행(자주)→실패 대응의 3활동이며, 실패를 즉시 고치는 것이 가장 중요하다.

- 이점: 디버깅 감소, 변경 자신감(리팩토링 장려), 실행 가능한 문서, 간단한 리뷰, 사려 깊은 설계(테스트 어려우면 설계 결함 신호), 빠른 고품질 릴리스.
- 테스트를 코드로 표현하면 모듈화·다양한 환경(브라우저/언어) 실행이 쉬움.

---

## 3. Test Size & Scope (크기와 범위)
> 크기는 자원(프로세스·머신·네트워크)을, 범위는 검증하는 코드 경로를 가리키며 둘은 별개다.

```
Test Size Constraints
Small  : single process (often single thread); NO sleep/IO/network/disk
         -> fastest, most deterministic (use test doubles)
Medium : single machine; multi-process, threads, blocking calls,
         network to localhost ONLY
Large  : multi-machine / remote cluster; e2e, config validation
```

- Small을 항상 우선 — 속도·결정성(determinism)이 가장 중요. Flaky test(비결정적 실패)는 매우 비싸다(1% 넘으면 신뢰 상실; Google은 ~0.15% 유지).
- 모든 테스트는 hermetic(자족적)이고, 조건문·반복문을 피해 "검사하면 자명(obvious upon inspection)"해야 한다.

### Test Scope & Pyramid (범위와 피라미드)
```
Test Pyramid (by test count)
        /\        ~5%  end-to-end (system)
       /  \      ~15%  integration (medium scope)
      /____\     ~80%  unit (narrow scope, small)
Antipatterns: "ice cream cone" (e2e 과다), "hourglass" (integration 부족)
```

- narrow scope(unit)는 빠르고 안정적이며 실패 진단이 쉬움; 큰 테스트는 sanity check이지 주 버그탐지 수단이 아님.
- 단, 비율은 팀의 아키텍처·조직 현실에 맞게 조정.

---

## 4. Beyoncé Rule, Coverage, Scale (베이온세 규칙·커버리지·규모)
> "If you liked it, then you shoulda put a test on it" — 깨지면 안 되는 것은 모두 테스트하라(실패 상황 포함).

- 실패 테스트: 예외/RPC 오류/지연 주입, Chaos Engineering으로 장애 대응을 능동적으로 검증.
- 코드 커버리지: 라인 실행 여부만 측정(결과 검증 아님) — 목표(예: 80%)가 천장이 되는 함정; 단일 숫자보다 "행동이 검증되는가"를 비판적으로 사고하라.
- Google 규모: 단일 모노레포(monorepo) 20억+ 줄, 주당 ~2500만 줄 변경, 브랜치 없이 head에 커밋, TAP(Test Automated Platform) CI로 관리.

### Pitfalls of Large Test Suites (대형 스위트의 함정)
- Brittle tests(과도 명세·mock 남용)는 무관한 변경에도 깨져 리팩토링을 저해("no more mocks!" 정서).
- 느린 테스트(sleep/setTimeout 남용)는 실행 빈도를 낮춤 — 폴링·타임아웃으로 대체; 테스트를 프로덕션 코드처럼 존중·투자하라.

---

## 5. History & Limits (역사와 한계)
> 2005~2006 테스트 혁명은 명령(mandate)이 아니라 성공 시연과 문화 전파로 이뤄졌다.

- Testing Grouplet의 3대 이니셔티브: Orientation Classes(신입 교육·트로이 목마), Test Certified(5단계 성숙도 프로그램), Testing on the Toilet(화장실 1페이지 뉴스레터).
- 오늘날: 모든 변경은 코드+테스트 포함 리뷰 필수, Project Health(pH) 도구가 1~5점으로 자동 측정.
- 명령하지 않은 이유: 성공한 아이디어는 스스로 퍼진다 — 강제는 Google 문화에 역행.
- 자동화의 한계: 검색 품질·오디오/비디오 품질·보안 취약점 탐색 등은 인간 판단(Exploratory Testing)이 우수 — 발견 후엔 자동 테스트로 회귀 방지.

---

## Summary (핵심 정리)
- 자동화 테스트는 소프트웨어 변화를 가능하게 하는 토대이며, 확장하려면 반드시 자동화되어야 한다.
- 균형 잡힌 테스트 스위트(크기·범위의 적절한 혼합)가 건강한 커버리지를 유지하고, "깨지면 안 되는 것은 테스트하라".
- AI 시대 코드·변경 폭증 속에서 안전망을 설계하는 테스트 문화가 빠른 적응과 제품 신뢰의 시니어 레버리지다 — 문화 변화에는 시간이 걸린다.
