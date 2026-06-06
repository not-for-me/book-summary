# 16. Version Control and Branch Management

## 챕터 개요 (3줄 요약)
- 버전 관리(VCS)는 시간이라는 차원을 명시적으로 다루는 도구로, 소프트웨어 엔지니어링(시간에 걸친 프로그래밍)의 핵심이며 팀 협업을 선형 이하 노력으로 확장시킨다.
- Google은 5만 명이 공유하는 모노레포(Piper)와 "단일 진실 원천(Source of Truth)" + "One-Version Rule"을 통해 trunk 기반 개발을 실현한다.
- AI 시대에 코드가 폭증할수록, 의존성 버전 선택지를 없애고 장기 dev 브랜치를 피하는 단순한 브랜치 정책이 조직 효율의 시니어 레버리지가 된다.

---

## 1. What Is Version Control? (버전 관리란)
> VCS는 파일시스템(filename->contents)을 (filename, time, branch)->contents로 확장한 시스템이다.

- VCS의 핵심 기능: 원자적 커밋(atomicity), 마지막 sync 추적, 분기/병합(branch-and-merge) 자동화, "어느 것이 최신인가" 해결.
- 중요성: 프로그래밍(즉각적 코드 생산)과 엔지니어링(시간에 걸친 유지)의 구분 — VCS는 시간과 raw source의 상호작용을 관리하는 일차 도구.
- 부가 효과: 감사·규제 기록, 커밋 시점의 성찰(체크리스트·정적분석·테스트 실행 트리거).

---

## 2. Centralized vs Distributed VCS (중앙집중 vs 분산)
> 현대 VCS는 핵심 기능이 동일하며 차이는 주로 UX다 — 단, 중앙집중/분산이라는 아키텍처 차이가 정책·확장에 영향.

- Centralized(RCS→CVS→Subversion→Perforce): 단일 중앙 저장소, trunk가 곧 현재 버전.
- Distributed/DVCS(Git, Mercurial): "어디에 커밋할 수 있나?" — 모든 clone이 저장소; 오프라인·협업에 강하고 OSS에서 지배적.
- 실무적으로는 대체로 호환 — DVCS도 대개 정책상 중앙 저장소(GitHub trunk)를 둠.
- Google은 거대 규모(5만 엔지니어·수천만 커밋)와 Hyrum's Law 때문에 자체 중앙집중 VCS(Piper) 사용 — DVCS의 전체 다운로드 모델은 규모상 낭비.

---

## 3. Source of Truth (단일 진실 원천)
> 중앙집중 VCS는 trunk를 진실 원천으로 내장하지만, DVCS는 명시적 정책이 필요하다.

- 명확한 진실 원천이 없으면: "Presentation v5 - final - redlines" 혼돈 — 릴리스에 어떤 기능이 포함됐는지 선형 이하 노력으로 보장 불가.
- 진실 원천은 상대적·계층적일 수 있음(RedHat 엔지니어 vs Linus의 Source of Truth) — 선택·불확실성이 없으면 확장 문제 회피.
- 기술 기본값인 trunk 위에 조직의 정책·관례가 동등하게 중요하다.

---

## 4. Branch Management (브랜치 관리)
> 진행 중인 작업(work-in-progress)도 사실상 하나의 브랜치다.

- Dev Branches: 제품 안정성을 위한 장기 dev 브랜치는 근본적으로 잘못 — 작은 병합 > 큰 병합, 저자 병합 > 일괄 병합; CI·테스트·코드리뷰로 안정성 확보가 더 낫다.
- dev 브랜치 중독: "병합이 위험" → "병합을 늦추고 조율"로 가면 Merge Coordinator·병합 회의 같은 확장 불가 오버헤드 발생.
- 대안: trunk 기반 개발 + CI + 빌드 green 유지 + 미완성 기능 런타임 비활성화.
- Release Branches: dev 브랜치와 달리 대체로 양성(결국 폐기) — cherry-pick 최소화; CD를 달성한 최고 조직은 release 브랜치가 거의 없음.

```
DORA findings (Accelerate / State of DevOps)
trunk-based development + no long-lived dev branches
   => strong predictor of high-performing software orgs
```

---

## 5. Version Control at Google & One Version (Google의 VCS와 One Version)
> 5만 엔지니어가 단일 모노레포(Piper, 80TB+, 하루 6~7만 커밋)를 공유하며 OWNERS로 세분화된 소유권을 강제한다.

- One-Version Rule(핵심): "개발자는 '어떤 버전의 컴포넌트에 의존할까'라는 선택을 절대 가져선 안 된다".
- 위반 시나리오: 공통 라이브러리(Abseil 등)를 이름 변경 없이 fork하면, 한 바이너리에 두 버전이 링크되어 빌드 실패/런타임 버그 — shading은 함수엔 되지만 타입엔 안 됨.
- 선택지 제거가 개인엔 불편해도 조직 확장엔 일관성("choke point")으로 결정적.
- (거의) 장기 브랜치 없음: 보류 작업도 작은 단위로 trunk에 자주 커밋; 장기 dev 브랜치는 드물고 비싸야 함(시간 차원 의존은 매우 고비용; "build horizon" 6개월 캡).

---

## 6. Monorepos & Future (모노레포와 미래)
> 모노레포는 One Version 준수를 쉽게 만들지만 만능은 아니다.

- 이점: 진실 원천 자명, 도구·최적화의 일관된 확장, 코드 가시성으로 설계 정보 공유.
- 모노레포가 유일한 정답은 아님 — 비밀·법적·프라이버시 요구가 프로젝트마다 다르면 manyrepo 장점도 있음.
- 핵심은 모노레포 자체가 아니라 "One-Version 원칙 최대 준수"; VMR(Virtual MonoRepo, Git submodules·Bazel external deps·CMake subprojects)로 절충 가능.
- 미래 전망: VCS는 더 큰 저장소 성능 확장 + 저장소 간 연결(VMR) 양방향으로 진화; OSS도 호환 버전 묶음(virtual monorepo)으로 이동 예상 — "버전 번호는 타임스탬프"이며 version skew는 시간 차원의 비용.

---

## Summary (핵심 정리)
- 1인·1회성 토이를 넘는 모든 프로젝트에 버전 관리를 쓰고, "어느 버전에 의존할까" 선택지가 생기면 확장 문제가 발생한다.
- One-Version Rule은 조직 효율에 놀랍도록 중요하며, shading 등 우회는 순수 손실 노동이다.
- trunk 기반 개발이 고성과 조직의 예측 인자이고 장기 dev 브랜치는 좋은 기본값이 아니다 — fine-grained 저장소를 쓰더라도 의존성은 unpinned/trunk 기반으로(VMR) — AI 시대 조직 효율의 시니어 레버리지.
