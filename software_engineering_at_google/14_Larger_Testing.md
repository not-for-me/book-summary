# 14. Larger Testing

## 챕터 개요 (3줄 요약)
- 단위 테스트로 메울 수 없는 충실도(fidelity) 격차를 메우기 위해 더 큰 테스트가 필요하며, 이는 시스템 전체가 의도대로 동작함을 확인하는 위험 완화 전략이다.
- 큰 테스트는 SUT(System Under Test)·데이터·액션·검증으로 구성되며, 충실도와 비용/신뢰성 사이의 트레이드오프로 SUT 규모를 정한다.
- AI 시대에 시스템이 복잡해질수록, 위험을 식별하는 테스트 전략과 마찰을 최소화한 큰 테스트 설계가 시니어의 핵심 역량이 된다.

---

## 1. What Are Larger Tests? (큰 테스트란)
> 큰 테스트는 small test 제약(단일 스레드·프로세스·머신)에서 자유로워, 느리고·비밀폐(nonhermetic)·비결정적일 수 있다.

- 존재 이유는 Fidelity(충실도): 테스트가 실제 시스템 동작을 얼마나 반영하는가 — 단위 테스트는 프로덕션과 다르게 패키징됨.
- 단위 테스트의 격차: 불충실한 더블(unfaithful/stale doubles), 설정 문제(Google 대형 장애 1위 원인), 부하 시 문제, 예상 못한 입력/부작용, 창발적 행동과 "진공 효과(vacuum effect)".

---

## 2. Why Not / Larger Tests at Google (왜 안 쓰나·역사)
> 좋은 테스트는 신뢰성·속도·확장성을 갖춰야 하는데, 큰 테스트는 이 셋을 종종 위반한다.

- 추가 도전: 소유권(여러 유닛/팀에 걸침 → 방치 시 부패), 표준화 부재(팀마다 방식 상이 → LSC에서 누락).
- 역사: Google은 단위 테스트 이전부터 대형 e2e 테스트 사용(2001 AdWords e2e 등); TAP는 밀폐 단위 테스트만 수용해 큰 테스트는 별도 처리.
- 시간 영향: "ice cream cone" 안티패턴 — 수동 테스트로 시작하면 며칠 만에 "레거시 코드" 발생; 초기 며칠 내 테스트 피라미드로 이동 필요.

---

## 3. Scale & Structure (규모와 구조)
> 시스템이 커지면 e2e 시나리오가 지수적/조합적으로 증가해 확장 불가 — 그러나 저충실도 더블의 버그 확률도 지수적이다.

- 예: 두 개의 10% 정확도 더블만 써도 버그 확률 99%(1 - 0.1*0.1). 높은 충실도를 유지하며 큰 테스트 구현이 중요.
- "Smallest possible test": 큰 테스트도 작을수록 좋음 — 출력을 데이터 저장소에 보존해 페어와이즈로 "체이닝(chaining)".
- 큰 테스트 워크플로 4단계: SUT 확보 → 테스트 데이터 시딩 → 액션 수행 → 행동 검증.

### System Under Test (SUT)
```
SUT forms (low fidelity -> high fidelity; high hermeticity -> low)
Single-process (small)  -> Single-machine (medium)
  -> Multimachine (large) -> Shared staging/production
  -> Hybrids (own binaries + shared backends)
Two opposing factors: Hermeticity vs Fidelity
```

- 밀폐 SUT의 이점: 충돌·예약 없이 코드 릴리스 전 실행 가능(production 테스트는 너무 늦음).
- 경계에서 SUT 축소: UI/API 경계, 서드파티 의존성에서 테스트 분할; in-memory DB로 대체.
- Record/Replay 프록시: 큰 테스트에서 외부 서비스 트래픽을 기록(Record mode, post-submit)하고 작은 테스트에서 재생(Replay mode, presubmit) — 비결정성 때문에 matcher로 요청 매칭.

### Test Data & Verification
- 데이터: 시드 데이터(domain data·현실적 baseline) + 테스트 트래픽; 생성 방식은 수작업·복사·샘플링("smart sampling").
- 검증: Manual(확장 안 됨), Assertions(명시적 체크), A/B 비교(diff, 두 SUT 출력 비교, 사람이 차이 검토).

---

## 4. Types of Larger Tests (큰 테스트 유형)
> SUT·데이터·검증 조합으로 다양한 유형이 만들어지며, 테스트 전략이 위험 벡터별로 필요한 테스트를 식별한다.

- 기능 테스트(하나 이상의 바이너리), 브라우저/기기 테스트.
- 성능·부하·스트레스 테스트(creating noise 제거 위해 동일 머신 배포 등 calibration).
- 배포 설정 테스트(설정 파일 smoke test), 탐색적 테스트(Exploratory, bug bash).
- A/B diff 회귀 테스트: Google에서 가장 흔한 큰 테스트(2001~) — old/new 응답 비교; 도전: 승인(approval)·노이즈·커버리지·셋업.
- UAT(사용자 인수 테스트, "의도대로" 검증), Probers/Canary 분석(프로덕션 헬스 모니터링).
- 재해 복구/카오스 엔지니어링: DiRT(연례 대규모 장애 주입), Catzilla(주당 수천 카오스 테스트).
- 사용자 평가: Dogfooding, Experimentation(A/B 실험 — AdWords 음영색 실험 일화), Rater evaluation(ML 시스템 평가).

---

## 5. Developer Workflow (개발자 워크플로)
> 큰 테스트도 presubmit/post-submit 자동화로 워크플로에 통합하되, 마찰을 최소화해야 한다.

- 대부분 TAP에 안 맞아 별도 post-submit 연속 빌드; presubmit 실행으로 저자에게 직접 피드백.
- 작성: 명확한 라이브러리·문서·예제 필요; A/B diff/production SUT는 유지비가 낮아 인기.
- 속도 향상: 범위 축소·분할·병렬화; sleep/timeout 대신 폴링(microsecond)·이벤트 핸들러·알림 구독; 내부 타임아웃 튜닝.
- Flakiness 제거 최우선: 범위 축소(hermetic SUT)·반응형 설계; 실패 모드를 명확히.
- 이해 가능성: 명확한 실패 메시지("10개 기대했으나 1개"), 근본원인 추적(Dapper로 RPC 체인에 request ID 연관), 지원/연락처 제공.
- 소유권: 문서화된 소유자 필요(OWNERS, per-test annotation) — 없으면 테스트가 부패.

---

## Summary (핵심 정리)
- 큰 테스트는 단위 테스트가 다룰 수 없는 것을 커버하며, SUT·데이터·액션·검증으로 구성된다.
- 좋은 설계는 위험을 식별하는 테스트 전략과 그것을 완화하는 큰 테스트를 포함한다.
- 큰 테스트는 충실도를 유지하면서도 최대한 작게 만들어 개발 워크플로의 마찰을 줄여야 한다 — AI 시대 복잡한 시스템의 시니어 레버리지.
