# 09. Code Review

## 챕터 개요 (3줄 요약)
- Google은 거의 모든 변경을 커밋 전(precommit)에 리뷰하며, LGTM(looks good to me) 승인을 핵심 권한 비트로 사용한다.
- 코드 리뷰는 정확성 검증을 넘어, 이해 가능성·일관성·집단 소유 의식·지식 공유·역사적 기록이라는 더 미묘한 이점을 제공한다.
- AI 시대에 코드 생산이 폭증할수록, 작은 변경·빠른 피드백·자동화로 리뷰를 가볍게 유지하는 것이 시니어가 코드 건강을 지키는 레버리지다.

---

## 1. Code Review Flow (리뷰 흐름)
> 작성자가 diff 스냅샷을 올려 리뷰어에게 보내고, 코멘트→수정 반복 후 LGTM을 받아 커밋하는 precommit 프로세스다.

- 단계: 변경 작성 → 스냅샷 업로드(자가리뷰/자동 코멘트) → 리뷰어에게 발송 → 코멘트 → 수정·재업로드 반복 → LGTM → 코멘트 해결·승인 후 커밋.
- 핵심 명제: "Code is a liability(코드는 부채다)" — 새 코드는 비행기 연료처럼 무게가 있다; "처음부터 새로 짜면 잘못하고 있는 것".
- 리뷰는 과거 설계 결정을 재논의하는 자리가 아니다(설계 리뷰는 별도).

---

## 2. How It Works at Google (세 가지 승인)
> 변경에는 세 종류의 승인이 필요하며, 한 사람이 셋을 겸할 수 있어 대부분 빠르게 진행된다.

- LGTM: 다른 엔지니어가 정확성·이해 가능성을 확인.
- Owner approval: OWNERS 파일에 등재된 디렉터리 소유자(게이트키퍼/스튜어드)가 해당 코드 영역에 적절한지 승인.
- Readability approval: 해당 언어 readability 인증자가 스타일·베스트프랙티스 부합을 확인.
- 역할 분리는 유연성과 확장(scale)을 위함 — 보통 2단계(동료 LGTM → 오너/readability 승인).

```
OWNERS hierarchy (additive up the tree)
/                 <- root owners (global approvers for LSC)
/foo/OWNERS       <- owns foo and children
/foo/bar/OWNERS   <- owns bar; effective owners = union up the tree
```

---

## 3. Code Review Benefits (이점)
> 정확성 검증은 1차 이점일 뿐, 더 큰 가치는 이해 가능성·일관성·문화·지식 공유에 있다.

- Code Correctness: 결함을 일찍 잡으면(shift left) 비용 절감 — 단, 정적분석·자동테스트가 인간 리뷰를 보완.
- Comprehension: 저자 편향 없는 첫 외부 독자 — 코드 이해 질문은 "고객은 항상 옳다"로 대하라.
- Consistency: 일관된 코드는 이해·유지보수·자동 리팩토링이 쉽다(때로 기능성보다 단순성 우선).
- 심리·문화: 코드는 "내 것"이 아니라 집단 자산 — 리뷰가 "나쁜 경찰" 역할을 해 비판을 중립화하고 imposter syndrome 검증도 제공.
- Knowledge Sharing: 시의적절·실행 가능한 양방향 지식 전수(엔지니어들이 리뷰로 처음 "만나기"도 함).
- 역사적 기록: 모든 변경이 코드베이스 일부가 되어 패턴 도입 시점을 추적 가능.

---

## 4. Best Practices (베스트프랙티스)
> 리뷰를 가볍고 빠르게 유지하는 것이 확장의 핵심이다.

- Be Polite and Professional: 리뷰어는 저자 접근법을 존중하고 결함 있을 때만 대안 제시; 24시간 내 피드백; 저자는 "당신은 당신의 코드가 아니다"를 기억하고 PTAL로 civil하게 대응.
- Write Small Changes: 약 200줄 이하 — 이해·롤백·버그추적 쉬움; 대부분 1일 내 초기 피드백, 35%는 단일 파일 변경.
- Write Good Change Descriptions: 첫 줄은 요약(prime real estate), "Bug fix"는 무용 — 무엇을 왜 바꿨는지 역사적 기록으로 상세히.
- Keep Reviewers to a Minimum: 대부분 리뷰어 1명 — 첫 LGTM이 가장 중요하고 추가 리뷰어는 한계효용 체감.
- Automate Where Possible: presubmit(테스트·린터·포매터·정적분석)로 기계적 작업 자동화 → 리뷰어가 본질에 집중.

---

## 5. Types of Code Reviews (리뷰 유형)
> 유형마다 초점이 다르다.

- Greenfield(신규 코드): 가장 드물고 가장 중요 — 설계 부합·완전한 테스트·OWNERS·CI 도입 확인.
- Behavioral Changes/Optimizations: 대부분의 변경 — 최고의 변경은 삭제(dead code 제거)일 수 있음; 테스트·벤치마크 갱신.
- Bug Fixes/Rollbacks: 버그만 집중(범위 확장 금지) + 회귀 테스트 추가; 롤백은 작고 원자적(atomic)이어야 안전.
- Refactorings/LSC(Large-Scale Changes): 기계 생성 변경도 리뷰하되, 도구·프로세스가 아닌 자기 코드 특화 우려만 표시하고 범위 확장 자제.

---

## Summary (핵심 정리)
- 코드 리뷰는 정확성·이해 가능성·일관성을 보장하고, 독자를 위해 최적화하며 가정을 타인을 통해 검증한다.
- 전문성을 유지한 비판적 피드백과 조직 전반의 지식 공유, 그리고 역사적 기록을 제공한다.
- 작은 변경·빠른 피드백·자동화가 프로세스 확장의 핵심 — AI 시대 코드 폭증 속 코드 건강을 지키는 시니어의 레버리지다.
