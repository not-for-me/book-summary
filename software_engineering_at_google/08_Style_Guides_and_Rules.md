# 08. Style Guides and Rules

## 챕터 개요 (3줄 요약)
- 규칙(rule)은 반드시 따라야 할 "법"이고 가이던스(guidance)는 권장 "should"이며, Google은 언어별 스타일 가이드를 정전(canon)으로 운영한다.
- 규칙의 목적은 "어떤 규칙을 둘까"가 아니라 "어떤 목표를 진전시킬까" — 시간과 규모에 대한 코드베이스 지속가능성과 복잡성 관리다.
- AI 시대에 코드가 폭증할수록, 가독성 최적화·일관성·자동화된 규칙 집행이 코드베이스를 유지보수 가능하게 지키는 시니어의 레버리지가 된다.

---

## 1. Why Have Rules? (왜 규칙인가)
> 규칙은 "좋은" 행동을 장려하고 "나쁜" 행동을 억제하며, 조직이 중시하는 가치에 따라 정의된다.

- 규칙은 코딩의 공통 어휘(common vocabulary)를 형성해, 엔지니어가 "어떻게 말할지"가 아니라 "무엇을 말할지"에 집중하게 한다.
- 결과적으로 기본값(default)으로 "좋은" 일을 하도록 부드럽게 유도한다.

---

## 2. Guiding Principles (지도 원칙)
> 3만+ 명, 20억+ 줄, 수십 년 수명의 코드베이스에서 규칙의 목표는 복잡성 관리와 생산성 유지의 트레이드오프다.

- 규칙은 제 몫을 해야 한다(Pull their weight): 자명한 것은 규칙화하지 않는다(예: C++의 goto 금지 규칙 없음).
- 독자를 위해 최적화(Optimize for the reader): 코드는 쓰이는 것보다 훨씬 자주 읽힌다 — "읽기 쉬움"을 "쓰기 쉬움"보다 우선.
  - 의도의 명시적 증거 남기기: override 키워드, 소유권 전달 시 std::unique_ptr + std::move로 지역적 추론(local reasoning) 가능.
- 일관성(Be consistent): 낯선 코드베이스에 빠르게 투입 가능; 도구화·확장·시간 회복탄력성을 가능케 함.
- 오류 유발·놀라운 구문 회피(Avoid error-prone/surprising): Python reflection(hasattr/getattr) 같은 파워 기능 제한 — 전문가뿐 아니라 모두가 다룰 수 있어야 함(SRE 디버깅 포함).
- 실용성에 양보(Concede to practicalities): 성능(noexcept), 상호운용성(snake_case 예외, Windows 다중상속)을 위해 예외 허용.

```
Consistency Hierarchy (local -> global)
file  >  team  >  project  >  overall codebase
- "Be consistent" starts locally
- internal consistency usually > external, BUT
- when interacting/open-sourcing, align with external standards
  (e.g. Python: 2-space -> 4-space; Starlark adopted 4-space)
```

---

## 3. What Goes in a Style Guide (스타일 가이드의 내용)
> 모든 규칙은 세 범주에 속한다: 위험 회피, 베스트프랙티스 강제, 일관성 보장.

- 위험 회피(Avoiding danger): static, lambda, 예외 처리, 스레딩, 상속 등 기술적 사유의 must/must-not.
- 베스트프랙티스(Enforcing best practices): 주석 규칙, 파일 구조, 네이밍, 포매팅(가독성), 새 기능에 대한 잠정적 안전 펜스.
- 일관성(Building consistency): 들여쓰기·임포트 순서 등 — "무엇을 골랐는가"보다 "골랐다는 사실"이 가치(끝없는 논쟁 탈출).
- 사례: std::unique_ptr는 처음엔 금지했다가, move semantics 적응 후 소유권 명시 이점이 커서 허용으로 전환.

---

## 4. Changing the Rules (규칙 변경)
> 스타일 가이드는 정적이지 않으며, 변경은 증거 기반·해결책 기반(solution-based) 프로세스를 따른다.

- 각 규칙에 결정의 근거(pros/cons)를 문서화 → 조건이 바뀌면 재평가 가능.
- 사례: Python CamelCase → snake_case 전환 (C++ 래퍼 중심에서 독립 Python 앱·오픈소스 상호작용 증가로).
- 프로세스: 커뮤니티 토론(언어별 메일링리스트) → Style Arbiters(언어 전문가 오너)가 트레이드오프로 합의(투표 아님) 결정.
- 예외(Exceptions): 규칙을 깨는 것이 더 이로울 때만 신중히 waiver 부여 (코드베이스 무결성 > 프로젝트 일관성).

---

## 5. Guidance & Applying the Rules (가이던스와 적용)
> 규칙이 "must"라면 가이던스는 "should" — primer, "Tip of the Week", <Language>@Google 101 교육 등 경험의 지혜를 정리한다.

- 규칙은 사회적(교육·코드리뷰·readability)으로도, 기술적(도구)으로도 집행되며, 가능하면 도구 자동화를 강하게 선호한다.
- 자동화 이점: 규모 확장, 인간 편향 최소화, 해석 일관성, 변경 시 누락 방지(C++ 규칙 약 90% 자동 검증 가능).
- Error Checkers: clang-tidy(C++), Error Prone(Java)로 deprecated API 경고+자동수정 → 준수 비용을 낮춤.
- Code Formatters: 로봇이 평균적으로 인간보다 낫다 — presubmit 검사로 강제(clang-format, gofmt, dartfmt, buildifier 등).
- 사례 gofmt: 설정 노브 없는 표준 포매터를 1일차부터 배포 → 리뷰의 포매팅 논쟁 제거, 머신 편집 코드가 사람 편집과 구분 안 됨(gofix 등 도구 생태계 가능).
- 단, 판단이 필요한 규칙("복잡한 템플릿 메타프로그래밍 회피")이나 사회적 규칙(CL 크기는 작게)은 인간 재량에 맡긴다.

---

## Summary (핵심 정리)
- 규칙과 가이던스는 시간·규모에 대한 회복탄력성을 지향하고, 데이터를 알아야 규칙을 조정할 수 있다.
- 모든 것을 규칙화하지 말되 일관성이 핵심이며, 가능하면 집행을 자동화하라.
- AI 시대 코드 폭증 속에서, 가독성·일관성·자동 집행이 코드베이스를 지속가능하게 유지하는 시니어의 핵심 레버리지다.
