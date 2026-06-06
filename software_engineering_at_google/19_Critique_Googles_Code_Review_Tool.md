# 19. Critique: Google's Code Review Tool

## 챕터 개요 (3줄 요약)
- Critique은 Google의 사내 코드리뷰 도구로, 코드 가독성·유지보수성 향상이라는 리뷰의 목표를 지원하고 코드베이스 진입을 게이트키핑한다.
- 설계 원칙은 단순성(Simplicity)·신뢰의 토대(Trust)·일반적 소통(Generic communication)·워크플로 통합 — 특히 단순성이 가장 큰 영향을 끼쳤다.
- AI 시대에 리뷰 시간이 곧 코딩 못 하는 시간이므로, 마찰을 줄이고 명확성을 높이는 리뷰 도구 설계가 조직 생산성의 시니어 레버리지가 된다.

---

## 1. Tooling Principles (도구 원칙)
> Critique은 Google의 리뷰 중심 개발 문화에서 형성된 원칙을 강조하도록 설계됐다.

- Simplicity: 불필요한 선택 없는 매끄러운 UI, 빠른 로딩, 핫키, 명확한 시각적 상태 표시.
- Foundation of trust: 리뷰는 속도 저하가 아니라 권한 부여 — 사소한 코멘트 반영을 재확인하지 않고 저자를 신뢰; 변경을 전사 공개.
- Generic communication: 복잡한 프로토콜 대신 일반적 코멘트 — 도구가 아무리 좋아도 소통은 사람이 한다.
- Workflow integration: Code Search·Cider(웹 IDE)·테스트 결과 등과 통합하되 링크 위주(임베딩 자제) — "Code Central" 통합 도구는 단순성 위해 포기.

---

## 2. Code Review Flow (리뷰 흐름)
> Critique 리뷰는 대개 커밋 전(precommit) 이뤄진다.

```
Critique 6 stages
1. Create change (snapshot upload -> auto analyzers run)
2. Request review (pick reviewers; presubmits run)
3. Comment (unresolved by default; published atomically)
4. Modify & reply (diff between any snapshots)
5. Approve (LGTM + Approval + 0 unresolved -> green)
6. Commit
```

- Notifications: 이벤트 알림 모델로 Chrome 확장·이메일 등 외부 도구가 빌드 — 관심사 분리(Critique은 핵심 도구에 집중).

---

## 3. Stage 1: Create a Change (변경 생성)
> 도구는 모든 단계를 지원하되 커밋의 병목이 되어선 안 된다.

- 저자가 리뷰어의 눈으로 diff·분석 결과를 미리 보게 해 오해 방지; 버그 링크(autocomplete).
- Diffing(핵심): 구문 강조, 교차참조(Kythe), intraline diffing(문자 단위), 공백 무시, 이동 감지(move detection), side-by-side/overlay 모드.
- Analysis Results: 스냅샷 업로드 시 분석기(Tricorder) 실행 — 상태 chip(빨강=강조, 노랑=진행중), 결함에 fix suggestion(미리보기·적용).
- Tight Tool Integration: Cider·Code Search·Rapid·Zapfhahn(커버리지) 등과 연결(FUSE 기반 클라우드 workspace로 가능).

---

## 4. Stage 2: Request Review (리뷰 요청)
> 규모가 커지면 적합한 리뷰어 찾기가 문제다.

- GwsQ: 팀 alias로 리뷰어 자동 배정(부하 분산·휴가 고려).
- 리뷰어 추천 요소: 코드 소유자, 최근 변경자(친숙도), 가용성(같은 시간대), GwsQ alias 설정.
- Presubmits(precommit hook): 이메일 리스트 추가, 자동 테스트(자원 집약적이라 스냅샷마다가 아닌 요청/커밋 시), 프로젝트 불변식 강제 — 실패 시 차단.

---

## 5. Stages 3-4: Commenting (코멘트)
> 코멘트는 변경 조회 다음으로 흔한 행동이며, 모두가 코멘트 가능하다.

- 파일별 "reviewed" 체크박스(저자 수정 시 해제); "Please fix" 버튼으로 분석 결과를 unresolved 코멘트화.
- "Done"/"Ack" 단축으로 빠른 응답; 코멘트는 작성 중 → 원자적으로 게시(publish).
- 상태 이해: "Whose turn" 기능(attention set — 변경이 현재 누구에게 막혀 있는가, 굵게 표시); Dashboard(Changelist Search 기반, 정규식 쿼리, 빠른 인덱싱).

---

## 6. Stages 5-6: Approval & Commit (승인과 커밋)
> 점수는 LGTM + Approval + unresolved 코멘트 수 3요소로 구성된다.

- LGTM("표준 충족, 커밋 OK") + Approval(게이트키퍼 허가) + unresolved 0 → 커밋 가능; 모든 변경에 최소 1 LGTM(두 쌍의 눈).
- 단순화된 평가: "Needs More Work"/"LGTM++" 제거 → LGTM/Approval은 항상 긍정 — 부정 피드백은 반드시 구체적(unresolved comment)이어야 함(문화에 긍정 영향).
- LGTM/Approval은 하드 요구(리뷰어만, 취소 가능), unresolved는 소프트(저자가 resolve) — 신뢰·소통에 의존.
- 커밋 버튼 내장(CLI 전환 회피).

### After Commit & Gerrit
- 변경 고고학(archaeology): 커밋 후에도 이력·코멘트 조회, 롤백 추적, 교육 자료 활용.
- Gerrit: Git 통합 오픈소스 리뷰 도구(Chrome·Android 등 모노레포 밖 프로젝트용) — 세분화 권한, 풍부한 플러그인, -2 veto 등 정교한 점수.

---

## Summary (핵심 정리)
- 신뢰와 소통이 코드리뷰의 핵심이며, 도구는 경험을 향상시킬 뿐 이를 대체할 수 없다.
- 다른 도구와의 긴밀한 통합이 훌륭한 리뷰 경험의 열쇠다.
- "attention set" 같은 작은 워크플로 최적화가 명확성을 높이고 마찰을 크게 줄인다 — AI 시대 조직 생산성의 시니어 레버리지.
