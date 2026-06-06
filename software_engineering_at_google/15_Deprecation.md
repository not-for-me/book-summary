# 15. Deprecation

## 챕터 개요 (3줄 요약)
- Deprecation(폐기)은 obsolete 시스템에서 질서 있게 이전하고 최종 제거하는 과정으로, 시간에 따른 시스템 관리라는 소프트웨어 엔지니어링의 본질에 속한다.
- 핵심 전제는 "코드는 자산이 아니라 부채(code is a liability, not an asset)" — 가치는 코드가 아니라 기능에서 나오며, 불필요한 코드 제거가 코드당 기능을 극대화한다.
- AI 시대에 코드가 폭증할수록, 폐기를 설계 단계부터 계획하고 정책·도구로 관리하는 능력이 장기 지속가능성의 시니어 레버리지가 된다.

---

## 1. Why Deprecate? (왜 폐기하는가)
> 코드는 생성·유지 비용을 수반하며, 노후 시스템 유지 vs 폐기의 트레이드오프를 평가해야 한다.

- 나이만으로는 폐기 근거가 안 됨(LaTeX처럼 오래되어도 obsolete가 아닐 수 있음) — 명백히 obsolete하고 동등 기능의 대체재가 있을 때 적합.
- 두 시스템 병존은 유지비 증가·상호 변환 코드·신시스템 진화 저해를 낳음.
- 핵심: 코드량이 아니라 "코드 단위당 전달 기능"을 극대화 — 불필요한 코드 제거가 가장 쉬운 방법.
- 단, 조직이 동시에 감당할 수 있는 폐기량에는 한계가 있음(도로를 전부 동시에 포장하면 아무도 못 감) — 집중과 완수 약속이 중요.

---

## 2. Why Is Deprecation So Hard? (왜 어려운가)
> Hyrum's Law: 사용자가 많을수록 예상치 못한 의존이 많아 제거(궁극의 변경)가 어렵다.

- 대체 시스템은 "더 낫지만 다름" — 일대일 매칭이 드물어 모든 사용처를 새 맥락에서 평가해야 함.
- 감정적 애착("I like this code!")과 change aversion — Google은 코드 삭제 후에도 이력 검색 가능(Chapter 17)으로 완화.
- 정치적·자금적 어려움: 폐기 비용은 가시적이나 방치 비용은 잘 안 보임 — 연구 기법(Chapter 7)으로 가치를 입증.
- 종종 in-place 점진적 진화가 전면 교체보다 저렴(전면 마이그레이션 비용은 자주 과소평가됨).

---

## 3. Deprecation During Design (설계 단계의 폐기)
> 폐기 용이성은 시스템을 처음 만들 때 설계 가능하다(원자력 발전소 해체 계획처럼).

- 핵심 질문: 소비자가 대체재로 얼마나 쉽게 이전할 수 있는가? 시스템을 점진적으로 교체할 수 있는가?
- 대부분의 소프트웨어는 이렇게 설계되지 않음(신제품 빠른 출시 문화의 disincentive).
- 핵심 원칙: "조직이 예상 수명 동안 지원할 의지가 없는 프로젝트는 시작하지 마라" — 시작 시점에 장기 지원 결정이 내려진다.

---

## 4. Types of Deprecation (폐기 유형)
> 폐기는 자문(advisory)부터 강제(compulsory)까지의 연속체다.

```
Deprecation Continuum
Advisory (no deadline, aspirational)
  - announces new system, encourages early adopters
  - "Hope is not a strategy" - rarely drives migration alone
  - works best when new system is TRANSFORMATIVE (not incremental)
        |
        v
Compulsory (has deadline + enforcement)
  - centralized expert team does migration work
  - needs enforcement power + active staffing
  - increasing-duration outages reveal hidden dependencies (like DiRT)
```

- Advisory: 데드라인 없음 — 새 시스템 광고·초기 채택자 유도용이나, 마이그레이션 대부분을 하지 못함; 기존 사용처가 개념적 "끌림"을 만듦.
- Compulsory: 데드라인+집행 메커니즘 필요 — 전문가 팀에 전문성 집중; 자금 없는 강제는 "unfunded mandate"로 마찰 유발.
- 의존성 발견: 모노레포로 가시성 확보 + 임시 turn-off/심볼 이름 변경으로 미상 의존 동적 발견.

### Deprecation Warnings (폐기 경고)
> 경고는 actionable(실행 가능)하고 relevant(시의적절)해야 한다 — 아니면 "alert fatigue".

- Actionable: 평균 엔지니어가 다음 단계를 실제로 수행 가능(예: 갱신된 함수로 교체).
- Relevant: 사용자가 해당 동작을 할 때 노출(코드 작성 시점, 마이그레이션 수개월 전).
- 모든 것에 경고를 붙이지 말 것 — ErrorProne/clang-tidy로 새로 변경된 라인에만 타깃 노출.

---

## 5. Managing the Process (프로세스 관리)
> 명시적 소유자 없이는 폐기가 진전되지 않는다.

- Process Owners: 전담 소유자 필요 — 없으면 advisory로 전락(끝나지 않음)하거나 영구 유지; 버려진(abandoned) 프로젝트는 전문가가 제거(20% time 활용).
- Milestones: 유일한 마일스톤을 "완전 제거"로 두지 말 것(외부엔 안 보임) — 측정 가능한 점진적 마일스톤을 축하.
- Tooling 3종:
  - Discovery: Code Search·Kythe로 정적 사용처 파악, 로깅·런타임 샘플링, 전역 테스트 스위트를 oracle로.
  - Migration: LSC(Large-Scale Change) 프로세스·코드 생성/리뷰 도구.
  - Backsliding 방지: Tricorder 정적분석(@deprecated 어노테이션, 리뷰 시 경고+자동수정), 빌드 visibility whitelist.

---

## Summary (핵심 정리)
- 소프트웨어는 지속적 유지비를 수반하므로 제거 비용과 저울질해야 하며, 제거는 보통 구축보다 어렵다(사용자가 설계 의도 밖으로 사용).
- in-place 진화가 보통 전면 교체보다 저렴하며(turndown 비용 포함), 폐기 비용·생태계 비용은 분산되어 측정이 어렵다.
- 폐기는 정책과 도구로 사회적·기술적 과제를 관리하는 과정 — AI 시대 코드 폭증 속 장기 지속가능성의 시니어 레버리지다.
