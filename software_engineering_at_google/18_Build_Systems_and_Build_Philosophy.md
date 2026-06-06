# 18. Build Systems and Build Philosophy

## 챕터 개요 (3줄 요약)
- 빌드 시스템의 목적은 소스를 실행 바이너리로 변환하며 빠르고(Fast) 정확하게(Correct) 하는 것 — Bazel(내부 Blaze)은 속도와 정확성의 양자택일을 피한다.
- 핵심 통찰은 빌드를 "태스크(task)"가 아니라 "아티팩트(artifact)" 중심으로 재구성하는 것 — 엔지니어의 자유를 제한해 시스템이 병렬화·캐싱·분산 빌드를 보장하게 한다.
- AI 시대에 코드·개발자가 많아질수록, 의존성을 세분화(fine-grained)하고 명시적으로 버전 관리하는 빌드 인프라가 생산성의 결정적 레버리지가 된다.

---

## 1. Purpose & Without a Build System (목적과 부재 시)
> 빌드 시스템은 개발을 확장시키며, 대부분의 빌드는 사람이 아닌 자동화(테스트·릴리스·CI)로 트리거된다.

- 컴파일러만으로는 다언어·다의존성·외부 라이브러리·순서·증분 빌드 처리 불가.
- 셸 스크립트는 점점 지루·느림·디버깅 고통·재현 불가 — 여러 개발자·머신 조율 시 붕괴(classic problem of scale).
- 자동화 워크플로: 자동 빌드·테스트·프로덕션 푸시, 코드리뷰 시 테스트, 머지 전 테스트, 저수준 라이브러리 전역 테스트, LSC.

---

## 2. It's All About Dependencies (의존성이 핵심)
> 자기 코드 관리는 쉽지만 의존성 관리가 어렵다 — "이게 있어야 저게 된다"가 빌드 시스템 설계의 본질.

- 태스크 의존(문서 푸시 후 릴리스), 아티팩트 의존(최신 라이브러리 필요), 내부/외부 의존 등.

---

## 3. Task-Based Build Systems (태스크 기반)
> 작업 단위가 "태스크"인 시스템(Ant, Maven, Gradle, Grunt, Rake) — buildfile에 태스크와 의존 그래프(acyclic) 정의.

- 어둠의 측면: 엔지니어에게 너무 많은 권한, 시스템엔 너무 적은 정보.
- 병렬화 어려움(B·C가 같은 자원 건드리는지 시스템이 모름 → 단일 스레드 강제 위험).
- 증분 빌드 어려움(태스크가 무엇이든 할 수 있어 재실행 필요 여부 판단 불가 → clean 습관화).
- 유지·디버깅 어려움(태스크도 코드; 비결정성·race condition·환경 가정 버그 빈발).

---

## 4. Artifact-Based Build Systems (아티팩트 기반)
> 시스템이 정의한 소수 태스크를 엔지니어가 제한적으로 설정 — buildfile은 "무엇을(what)" 선언, 시스템이 "어떻게(how)" 결정.

- 함수형 프로그래밍과 유사: 빌드는 소스+도구를 입력, 바이너리를 출력하는 수학 함수 → 병렬화·정확성 보장 용이.
- Bazel: BUILD 파일에 target(java_binary, java_library) 선언 — name, srcs, deps; workspace + WORKSPACE 파일.

```
Bazel build flow (bazel build :MyBinary)
1. parse all BUILD files -> dependency graph
2. compute transitive deps of MyBinary
3. build each dep in order (parallel where possible!)
4. build MyBinary linking everything
=> 2nd run with no change: "up to date" in <1s (output depends only on inputs)
```

- 추가 기법: 도구를 의존성으로 취급(toolchain으로 플랫폼 독립), 커스텀 rule로 확장(action = 최저 합성 단위), sandboxing(LXC)으로 환경 격리, 외부 의존성 결정성(manifest에 암호화 해시).

---

## 5. Distributed Builds (분산 빌드)
> 20억+ 줄, 수만 target 의존 — 단일 머신으로는 불가능, 작업을 다수 머신에 분산해야 한다.

- Remote Caching: 공통 원격 캐시(아티팩트를 target+입력 해시로 키) — 빌드 재현성이 전제; Google은 많은 아티팩트를 캐시에서 제공.
- Remote Execution: 빌드 마스터가 action을 worker 풀에 스케줄 — 환경 자기기술·자기완결·결정적 출력 필수(태스크 기반으론 거의 불가능).
- Google 구현: ObjFS(원격 캐시, Bigtable+FUSE objfsd, 온디맨드 다운로드로 2배 빠름), Forge(원격 실행, Distributor→Scheduler→Executor) — 매일 수백만 빌드.

---

## 6. Modules & Dependencies (모듈과 의존성)
> 모듈 구성이 성능과 유지보수에 큰 영향을 준다.

- Fine-grained modules & 1:1:1 Rule: 작은 모듈이 병렬·증분·테스트 선택 실행에 유리(프로덕션 바이너리가 수만 target 의존); Google은 자동 BUILD 관리 도구로 부담 완화.
- Minimize Visibility: 의존성의 반대 — 가능한 한 가시성 최소화(대부분 private).
- Internal deps: 소스에서 빌드, 버전 없음; "strict transitive dependency mode"로 직접 의존하지 않은 심볼 참조를 차단(빌드 가속).
- External deps: 버전 있음 — 자동(latest) vs 수동 버전 관리; 수동이 안정성으로 규모에서 가치(Bazel은 수동 강제).
- One-Version Rule: 서드파티 의존성에 단일 버전 강제 → diamond dependency 문제 방지.
- Transitive external deps: Bazel은 자동 다운로드 안 함 — 전역 파일에 모든 외부 의존성·버전 명시.
- 보안·신뢰성: 의존성 미러링, 해시 검증, vendoring(third_party에 체크인)으로 외부→내부 의존 전환.

---

## Summary (핵심 정리)
- 완전한 빌드 시스템은 조직 확장 시 개발자 생산성에 필수이며, 빌드 시스템을 적절히 제한하는 것이 오히려 개발자를 편하게 한다.
- 아티팩트 중심 빌드가 태스크 중심보다 잘 확장되고 신뢰성 높으며, 세분화된 모듈이 병렬·증분 빌드에 유리하다.
- 외부 의존성은 소스컨트롤 하에 명시적으로 버전 관리하라("latest" 의존은 재앙) — AI 시대 코드·개발자 폭증 속 생산성의 시니어 레버리지.
