# 23. Continuous Integration

## 챕터 개요 (3줄 요약)
- CI(Continuous Integration)의 근본 목표는 문제 있는 변경을 가능한 한 빨리 자동으로 잡는 것 — 현대적으로는 "복잡하고 빠르게 진화하는 전체 생태계의 지속적 조립·테스트"다.
- 핵심 개념은 빠른 피드백 루프, 자동화(Continuous Build/Delivery/Testing), 그리고 "언제 무엇을 테스트할지"(presubmit vs post-submit vs RC)의 균형이다.
- AI 시대에 시스템이 마이크로서비스로 분산되고 의존성이 외부화될수록, 비용을 왼쪽으로 옮겨(shift left) 일찍 문제를 잡는 CI 설계가 품질·생산성의 시니어 레버리지다.

---

## 1. CI Concepts (핵심 개념)
> 버그 비용은 늦게 잡힐수록 거의 지수적으로 증가하므로, 빠른 피드백 루프가 핵심이다.

- 피드백 루프(빠름→느림): edit-compile-debug → presubmit 자동 테스트 → post-submit 통합 에러 → staging QA → 내부 사용자 → 외부 사용자/언론.
- Canarying(일부 프로덕션 선배포)은 위험을 줄이나 version skew(분산 시스템에 비호환 버전 공존) 문제 유발.
- Experiments/Feature flags는 강력한 피드백 루프 — 모듈식 토글로 배포 위험 격리(CD의 흔한 패러다임).
- 피드백은 접근 가능(accessible)·실행 가능(actionable)해야 함 — 통합 테스트 리포트, flake 분류, 공개 버그.

### Automation: CB / CD / CT
- Continuous Build(CB): head의 최신 변경을 통합·빌드·테스트 → "green head"(CB 검증) vs "true head"(최신 커밋).
- Continuous Delivery(CD): release candidate(RC) 연속 조립(보통 green head에서) + 환경 단계별 승격·테스트; 설정(config)도 RC에 포함해 함께 테스트(설정 버그가 프로덕션 버그의 큰 비율).
- Continuous Testing(CT): 변경 수명 전반에 적용 — 오른쪽으로 갈수록 점점 큰 범위의 자동 테스트.

---

## 2. Presubmit vs Post-submit (언제 무엇을)
> presubmit만으로는 부족하다 — 모든 테스트를 presubmit에 돌리는 건 너무 비싸다.

```
Life of a code change (shift left = cheaper)
local edit -> presubmit -> post-submit(CB) -> RC(CD) -> staging -> production(probers)
- presubmit: fast & reliable tests only (usually unit tests, project-scoped)
- post-submit: longer, less stable tests OK (with failure management)
- "mid-air collision": two changes touching different files break a test
```

- RC 테스트(post-submit과 같은 스위트라도): sanity check, 감사성(auditability), cherry-pick 대응, 긴급 푸시.
- Production 테스트(probers): 프로덕션 동작 상태 + 테스트의 적절성 검증 — "defense in depth".

### CI Is Alerting (CI는 알림이다)
- CI는 워크플로 왼쪽(테스트 실패), 알림은 오른쪽(메트릭 임계치) — 둘 다 "가능한 빨리 문제 식별".
- flaky test = spurious alert: 실행 불가능하면 테스트 실패도 알림도 되어선 안 됨; 100% green은 100% uptime처럼 지나치게 비쌈.

---

## 3. CI Challenges (도전)
> presubmit 최적화, culprit finding/failure isolation, 자원 제약, failure management.

- 실패 관리: 큰 e2e 테스트는 깨지기·flaky 쉬움 — 버그 hotlist로 추적, release-blocker 즉시 수정; flaky 테스트는 일시 비활성화.
- 테스트 불안정성: 재시도(retry) 설정, hermetic testing.

### Hermetic Testing (밀폐 테스트)
> 외부 의존성 없는 자기완결 환경 — 결정성(stability)과 격리(isolation) 제공.

- presubmit에서 안정성 중요할 때 유용; 실패 시 원인이 코드/테스트 변경임을 알 수 있음.
- Fake(가짜 백엔드), 완전 sandbox 스택, record/replay(라이브 응답 기록·재생) — record/replay는 false positive(캐시 과다)/false negative(캐시 부족) 균형이 어려움.
- 사례 Google Assistant: presubmit을 완전 hermetic화 → 런타임 14배 단축, flakiness 거의 제거; hotswapping으로 N개 마이크로서비스 failure isolation을 O(N²)→O(N).

---

## 4. CI at Google (Google의 CI)
> TAP(Test Automation Platform): 전체 코드베이스의 거대 연속 빌드 — 하루 5만+ 변경, 40억+ 테스트 케이스.

- Presubmit Optimization: 잠재적 breaking 변경을 head에 진입 허용(presubmit 통과 변경의 95%+가 나머지 테스트도 통과) → post-submit에서 비동기로 영향 테스트 실행; 평균 제출 대기 ~11분.
- Build Cop: 프로젝트 테스트를 green으로 유지하는 책임 — 깨지면 즉시 롤백(선호) 또는 forward fix.
- Culprit Finding: 배치 분할 재실행 + binary search 도구; TAP은 confidence 높으면 자동 롤백.
- Resource Constraints: Forge(분산 빌드/테스트), 의존성 그래프로 영향받는 최소 테스트만 실행 — 작은 변경을 유도.

### Case Study: Google Takeout
- #1 깨진 dev 배포: 10+ 제품 API의 flag/config 충돌 → presubmit에 sandbox 미니 환경 + post-submit(2시간마다) e2e → 깨진 서버 95% 방지, 야간 배포 실패 50% 감소.
- #2 해독 불가 로그: 90+ 플러그인 실패 로그 → 파라미터화 테스트 러너 + 친화적 UI + 실패 메시지에 로그 링크 → 디버깅 관여 35% 감소.
- #3 "all of Google" 디버깅: 같은 스위트를 프로덕션에도 연속 실행 → failure isolation 저비용.
- #4 green 유지: 즉시 못 고치는 실패는 버그 태그로 비활성화(자동 정리, feature flag 인지) → 자가유지 스위트(MTTCU 지표).

---

## 5. But I Can't Afford CI? (CI 비용)
> 이미 프로덕션 문제 대응에 지불하는 비용을 생각하라 — CI는 새 비용이 아니라 더 이른(선호되는) 단계로 옮긴 비용이다.

- 잦은 프로덕션 소방은 스트레스·사기 저하; CI는 더 안정적 제품과 행복한 개발 문화로 이어짐.

---

## Summary (핵심 정리)
- CI 시스템은 어떤 테스트를 언제 쓸지 결정하며, 코드베이스가 노후·확장될수록 점점 필수가 된다.
- presubmit엔 빠르고 신뢰성 높은 테스트를, post-submit엔 느리고 덜 결정적인 테스트를 최적화하라.
- 접근 가능·실행 가능한 피드백이 CI를 효율적으로 만든다 — AI 시대 분산 시스템에서 비용을 왼쪽으로 옮기는 시니어 레버리지.
