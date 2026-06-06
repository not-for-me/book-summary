# 21. Dependency Management

## 챕터 개요 (3줄 요약)
- 의존성 관리는 우리가 통제하지 않는 라이브러리·패키지 네트워크를 시간에 걸쳐 관리하는 문제로, 소프트웨어 엔지니어링 정책에서 가장 어렵고 중요한 문제다.
- 핵심 난제는 단일 의존성이 아니라 "변화하는 의존성 네트워크"이며, 충돌하는 요구사항(diamond dependency) 문제를 비조율 그룹 간에 어떻게 피하는가다.
- AI 시대에 소프트웨어가 거대한 의존성 기둥 위에 세워질수록, SemVer의 한계를 이해하고 테스트·CI 기반 증거로 안전성을 판단하는 능력이 시니어의 핵심 통찰이다.

---

## 1. Why So Difficult? (왜 어려운가)
> 핵심은 단일 의존성이 아니라 시간에 따라 변하는 네트워크를 관리하는 것이다.

- 강력한 조언: "다른 조건이 같다면 의존성 관리보다 소스컨트롤 문제를 선호하라" — 조직을 더 넓게 정의(전사 모노레포)할 수 있다면 좋은 트레이드오프.
- Diamond Dependency: libbase를 liba·libb가 쓰고 libuser가 둘을 쓸 때, libbase가 비호환 변경하면 liba는 신버전·libb는 구버전에 의존 → 통합 불가.

```
Diamond Dependency Problem
        libuser
        /     \
     liba     libb
        \     /
        libbase   <- if incompatible change here, conflict!
- C++: near-zero tolerance (One Definition Rule, UB)
- Java: shading can rename symbols (partial; fails for shared types)
```

---

## 2. Importing Dependencies (의존성 가져오기)
> 재사용은 건강하지만, 시간을 고려하면 유지보수 비용이라는 숨은 트레이드오프가 생긴다.

- Compatibility Promises(호환성 약속) 사례: C++ 표준 라이브러리(거의 무한 후방호환, ABI 포함), Go(소스 호환만, 바이너리 비호환), Abseil(API 호환 + 자동 리팩토링 도구 제공, ABI 비약속), Boost(버전 간 호환 비약속, 실험적 proving ground).
- 핵심: 이는 프로그래밍이 아니라 소프트웨어 엔지니어링 문제 — "동작하게 만들었다" vs "지원되는 방식으로 동작한다"(Hyrum's Law).
- 가져오기 전 질문: 테스트가 있고 통과하는가? 누가 제공하나? 어떤 호환성을 지향하나? 인기·수명·breaking change 빈도? 내부 구현 비용·업그레이드 주체.

### How Google Handles It
- 대부분 의존성은 내부 개발(=소스컨트롤). 외부는 third_party 디렉터리에 추가.
- 문제 시나리오: Alice가 데모용으로 OSS를 third_party에 추가 → 검색 인프라가 transitive 의존 → 보안 취약점으로 긴급 업그레이드 필요 시 아무도 경험·소유권 없음 → 비싸고 어려운 업그레이드. third_party 정책이 이 흔한 시나리오에 부족.

---

## 3. Dependency Management, In Theory (이론)
> 좋은 해법은 동적 생태계에서도 충돌 요구사항을 피해야 하며, 4가지 흔한 접근이 있다.

- Nothing Changes(정적 모델): 가장 단순, 신규 조직엔 적합하나 충분히 긴 시간엔 거짓 — 보안 버그 등 강제 업그레이드 대비 불가.
- Semantic Versioning(SemVer): de facto 표준 — major(breaking)/minor(additive)/patch — SAT-solver로 버전 만족; 만족 불가 시 "dependency hell".
- Bundled Distribution(번들 배포): 배포자(distributor)가 상호 호환 버전 집합을 묶어 단일 단위로 릴리스(Linux 배포판).
- Live at Head: 의존성 unpin·SemVer 폐기·제공자가 전체 생태계 테스트 후 커밋 — trunk-based development의 의존성 확장; CI·테스트 전제.

---

## 4. The Limitations of SemVer (SemVer의 한계)
> SemVer 버전 번호는 "이 변경이 위험한가"에 대한 손실 압축(lossy) 추정치다.

- Overconstrain(과제약): libbase의 미사용 함수 Bar만 바뀌어도 major bump → SAT-solver가 호환 버전을 거부(불필요한 dependency hell).
- Overpromise(과약속): patch는 "안전"하다는 가정이 Hyrum's Law와 충돌 — 임포트 순서·로그 포맷 변경 등 관찰 가능한 모든 동작에 누군가 의존.
- 핵심: 변경은 고립적으로 breaking/non-breaking이 아니라 "어떻게 소비되는가"의 맥락에서만 평가 가능 — SemVer엔 그 정보가 없음.
- 동기(motivations): 유지보수자가 breaking change 비용의 극히 일부만 부담 → 안정 코드 유인 부족; Go/Clojure는 major bump를 새 패키지로 취급.
- Minimum Version Selection(MVS): liba가 ≥1.7 요구 시 최신(1.8) 대신 1.7을 선택 → 저자가 실제 테스트한 버전에 최대한 가깝게(고충실도).

### So, Does SemVer Work?
- 잘 작동 조건: 제공자가 정확·책임감 있고, 의존성이 세분화되어 있고, 모든 API 사용이 예상 범위 내일 때 — 소수의 잘 관리된 의존성엔 적합.
- 규모가 커지면 충실도 손실이 지배적 → false positive(이론상 호환 불가하나 실제 동작)와 false negative(동작하나 SAT-solver가 거부) 발생.

---

## 5. With Infinite Resources & Exporting (무한 자원과 내보내기)
> 무한 컴퓨트가 있다면 SemVer 추정 대신 "영향받는 다운스트림 테스트 실행"이라는 증거로 안전성을 판단 — 사실상 Live at Head.

- 필요한 변화: 모든 의존성에 단위 테스트, 의존성 네트워크의 역방향 엣지(dependents) 인덱싱, CI 컴퓨트 자원, unpin된 의존성, 이력·평판 반영.
- 변경 종류(Chapter 12)별로 위험을 추정해 적절한 테스트 수준 적용(순수 리팩토링은 가볍게, 동작 변경은 광범위하게).

### Exporting Dependencies (의존성 내보내기)
> 코드를 의존성으로 공개하는 것은 자선·기회만이 아니라 평판·효율 리스크를 동반한다("community over code").
- gflags 사례: 대규모 리팩토링 불가·라이선스 분리·기여 통합 불가로 내부/외부 fork 분기 → 방치되어 평판 손상; 공통 API에 의존하던 팀들이 Hyrum's Law로 깨짐.
- AppEngine 사례: Python/C++ 업그레이드를 일부 유료 고객이 거부 → 의존성 전이 폐쇄(transitive closure)가 3년간 구버전 컴파일러에 묶임 — 외부 사용자가 내부 사용자보다 훨씬 비싸다.

---

## Summary (핵심 정리)
- 의존성 관리보다 소스컨트롤 문제를 선호하고, 의존성 추가는 지속적 지원 비용을 이해하고 신중히 하라 — 의존성은 give-and-take 계약이다.
- SemVer는 "사람이 생각하는 변경 위험"의 손실 압축 추정이며, SAT-solver가 이를 절대값으로 격상해 과제약/과소제약을 낳는다 — 테스트·CI는 실제 증거를 제공하고 MVS는 고충실도다.
- 단위 테스트·CI·저렴한 컴퓨트는 의존성 관리의 패러다임을 바꿀 잠재력이 있다; 의존성 제공은 공짜가 아니다 — AI 시대 거대 의존성 기둥 시대의 시니어 통찰.
