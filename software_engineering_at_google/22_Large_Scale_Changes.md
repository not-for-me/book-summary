# 22. Large-Scale Changes

## 챕터 개요 (3줄 요약)
- 대규모 변경(LSC)은 논리적으로 연관됐지만 단일 원자적 커밋으로 제출 불가한 변경 집합으로, 코드베이스가 클수록 가능한 최대 원자적 변경 크기가 오히려 줄어든다.
- LSC는 마스터 변경을 작은 샤드로 쪼개 독립적으로 테스트·리뷰·커밋하며(Rosie), 인프라 팀이 중앙집중적으로 수행해 비용을 내부화한다.
- AI 시대에 코드가 자동 생성·변경될수록, 기술적 결정의 불변성을 재고하고 코드베이스를 유연하게 유지하는 LSC 역량이 인프라 확장의 시니어 레버리지다.

---

## 1. What Is an LSC? & Who Deals with It (LSC란·누가)
> LSC는 단일 원자 단위로 제출 불가한 논리적 연관 변경(도구 한계·머지 충돌·분산 저장소).

- 범주: 안티패턴 정리, deprecated 기능 교체, 저수준 인프라 개선(컴파일러 업그레이드), 구→신 시스템 이전.
- 대부분 거의 무영향(behavior-preserving 리팩토링); 수십만~수백만 참조 변경.
- 인프라 팀이 책임: 도메인 지식 보유, 에러 playbook, "unfunded mandate(자금 없는 명령)" 회피, 유인 정렬 — 유기적 마이그레이션은 실패하기 쉬움(개발자가 기존 코드를 예시로 사용).
- 사례 "Filling Potholes": stl_util.h vs map-util.h 구분자 통일 같은 작은 수정도 LSC 도구로 저비용 실행.

---

## 2. Barriers to Atomic Changes (원자적 변경의 장벽)
> 저장소·엔지니어가 커질수록 단일 원자 커밋은 비현실적이 된다.

- 기술적 한계: VCS 연산이 변경 크기에 선형 — 수천 파일 원자 커밋 불가, 다른 사용자 차단.
- 머지 충돌: 파일 수↑ × 엔지니어 수 → 충돌 확률 증가; 작은 변경이 충돌 적음.
- No Haunted Graveyards(유령 묘지 금지): 아무도 손 못 대는 고대 시스템 — LSC 진행을 막음; 철저한 테스트가 해법.
- Heterogeneity(이질성): VCS/CI/포매팅 다양성이 자동 변환을 방해 — 일관성(presubmit 표준화) 추구.
- Testing: 변경이 클수록 테스트·진단 어려움 — 작은 독립 변경이 검증·근본원인 파악 쉬움(작은 변경이 같은 테스트를 중복 실행하는 트레이드오프 수용).

### Case: Testing LSCs & TAP Train
> 프로젝트 변경의 10~20%가 LSC 결과 — 좋은 테스트 없이는 불가능.
```
TAP Train (every 3 hours)
1. each change: run 1,000 random tests
2. gather passing changes -> one "train" uber-change
3. run union of all affected tests (can be ALL tests, 6+ hours)
4. for each failing test: rerun per-change to find culprit
5. generate per-change report (evidence it's safe to submit)
* Insight: LSCs rarely interact; most tests pass for most LSCs
```

### Case: scoped_ptr → std::unique_ptr
- 50만+ 참조를 std::unique_ptr로 — type alias로 만든 뒤 텍스트 치환, 마지막에 alias 제거; 하루 700+ 독립 변경·15,000+ 파일.

---

## 3. LSC Infrastructure (인프라)
> 가장 중요한 지원은 LSC를 둘러싼 문화적 규범과 감독의 진화다.

- Policies & Culture: 모노레포·전사 가시성; 경량 승인 프로세스(경험 많은 엔지니어 위원회 감독); 로컬 소유자는 변경을 이해하되 LSC에 거부권 없음(신뢰 문화).
- Codebase Insight: Kythe(시맨틱 인덱싱, "이 함수 호출자?"), ClangMR·JavacFlume·Refaster(AST 변환); 변경 생성 도구는 코드베이스 크기에 대해 sublinear여야 함(500 edit 넘으면 도구 학습이 효율적).
- Change Management: Rosie — 마스터 변경을 프로젝트/소유권 경계로 샤딩, 독립적 test-mail-submit 파이프라인.
- Language Support: type aliasing·forwarding function이 비원자적 마이그레이션에 필수; 정적 타입 언어가 자동 변경에 유리(Python/Ruby/JS는 더 어려움); 자동 포매터(google-java-format/clang-format)가 핵심.
- 사례 Operation RoseHub: Apache Commons "Mad Gadget" 취약점에 50+ 자원자가 2,600+ 패치를 GitHub OSS 프로젝트에 — LSC 문화의 외부 확장.

---

## 4. The LSC Process (프로세스)
> 4단계: 승인 → 변경 생성 → 샤드 관리 → 정리.

- Authorization: 이유·영향·FAQ 문서 작성 → 위원회 검토(흔히 "global approver" 지정); 감독·에스컬레이션 경로 제공이지 금지가 목적 아님.
- Change Creation: 최대한 자동화(backslide·머지 충돌 대비); 사람이 읽을 수 있게(human-like, 스타일 가이드·포매터).
- Sharding & Submitting: Rosie가 샤딩 → 각 샤드 독립 test-mail-submit; 공유 인프라 부하 관리.
  - "Cattle vs Pets": 개별 변경은 이름 없는 소(cattle)처럼 — 머지 충돌 등으로 거부돼도 저비용 재생성.
  - Testing: 각 샤드를 transitive 영향 테스트로; flaky 테스트가 LSC 처리량에 큰 지장(최근 flaky 테스트는 자동 제출 시 무시).
  - Reviewing: 로컬 소유자가 LSC를 과신하는 경향 → 맥락 필요 시만 로컬, 나머지는 global approver가 패턴 기반 도구로 일괄 승인(확장).
- Cleanup: "done"의 정의는 다양(완전 제거 vs 고가치 참조만); Tricorder로 deprecated 재도입을 리뷰 시 차단(backslide 방지).

---

## Summary (핵심 정리)
- LSC 프로세스는 특정 기술 결정의 불변성을 재고하게 만든다 — 널리 쓰이는 심볼 이름·클래스 위치가 더는 최종이 아니다.
- 전통적 리팩토링 모델은 대규모에서 깨진다.
- LSC를 한다는 것은 LSC를 습관화하는 것이다 — AI 시대 코드 자동 변경 속 코드베이스 유연성을 지키는 시니어 레버리지.
