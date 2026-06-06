# 01. What Is Software Engineering?

## 챕터 개요 (3줄 요약)
- 소프트웨어 엔지니어링은 "시간에 걸쳐 통합된 프로그래밍"으로, 코드를 작성하는 행위(programming)와 그 코드를 수명 내내 유지·진화시키는 활동(engineering)을 구분한다.
- 핵심 3축은 시간(Time), 규모(Scale), 트레이드오프(Trade-offs)이며, 지속가능성(sustainability)은 필요한 변화에 안전하게 대응할 수 있는 능력으로 정의된다.
- AI 시대에 작성·실행 비용이 급감할수록, 문제를 어떻게 설정하고 장기적으로 유지 가능한 의사결정을 내리는가라는 "엔지니어링 그 자체"의 중요성이 더 커진다.

---

## 1. Time and Change (시간과 변화)
> 코드의 기대 수명(expected life span)은 수 분에서 수십 년까지 약 10만 배 차이가 나며, 수명이 길수록 변화 대응 능력이 본질적 과제가 된다.

- 단명(short-lived) 코드는 사실상 "프로그래밍 문제"지만, 장수(long-lived) 프로젝트는 OS(Operating System)·언어·라이브러리·하드웨어 변화에 반드시 반응해야 한다.
- 업그레이드를 처음부터 계획하지 않은 프로젝트는 첫 대규모 전환이 매우 고통스럽다: 숨은 가정이 누적되고, 경험이 없으며, 변경 규모가 크기 때문이다.
- "동작한다(happens to work)"와 "유지보수 가능하다(is maintainable)"를 구분하는 것이 장기 지속가능성의 핵심이다.
- AI 시대 관점: AI가 코드를 빠르게 생산할수록, 그 코드가 "수명 내내 변경 가능한가"를 설계 단계에서 판단하는 시니어의 안목이 차별점이 된다.

### Hyrum's Law (하이럼의 법칙)
> 충분히 많은 API(Application Programming Interface) 사용자가 있으면, 계약서에 무엇을 약속하든 시스템의 모든 관찰 가능한 동작은 누군가에게 의존된다.

- 엔트로피처럼 제거할 수는 없고 완화(mitigate)만 가능하다 — 변경·유지보수 논의의 지배적 요인이다.
- 예: 해시(hash) 순회 순서는 보장되지 않지만, 사용자는 결국 그 순서에 의존하는 코드를 작성한다.
- 함의: 단명 코드에서는 그런 의존이 무해하지만, 장수 프로젝트에서는 변경을 막는 리스크가 된다.
- 격언: "'clever'가 칭찬이면 프로그래밍, 'clever'가 비난이면 소프트웨어 엔지니어링이다."

```
Expected Life Span Spectrum (Figure 1-1 concept)
[minutes/hours] --- one-off script (change irrelevant)
       |
       |  <-- transition: must start reacting to external change
       v
[years/decades] --- Google Search / Linux kernel (must stay current)
```

---

## 2. Scale and Efficiency (규모와 효율)
> 조직이 성장할 때 반복 작업이 인력 투입 대비 선형(linear) 이하로 확장되어야 하며, 초선형(superlinear) 비용은 지속 불가능의 신호다.

- 코드베이스 지속가능성: 바꿔야 할 모든 것을 안전하게, 코드베이스 수명 내내 바꿀 수 있는 상태.
- 인력·컴퓨팅·코드베이스(빌드/버전관리) 모두 확장 가능해야 한다 — "삶은 개구리(boiled frog)"처럼 천천히 악화되는 문제를 경계.
- 확장 안 되는 정책 예: 사용자에게 마이그레이션을 떠넘기는 전통적 deprecation, 기능별 dev branch 남발.
- 잘 확장되는 정책 예: "Churn Rule"(인프라 팀이 직접 in-place 업데이트), "Beyoncé Rule"("CI(Continuous Integration) 테스트로 보호하지 않았다면 인프라 변경의 책임이 아니다").
- 전문성(expertise)과 공유 포럼은 규모가 커질수록 초선형 가치를 제공한다.

### Example: Compiler Upgrade (컴파일러 업그레이드)
- 2006년 Google의 첫 대규모 컴파일러 업그레이드는 극도로 고통스러웠다 — 5년간 미루며 Hyrum's Law 의존성이 누적됐기 때문.
- 교훈: 자주 바꿀수록 쉬워진다. 유연성을 높이는 요인은 전문성(Expertise)·안정성(Stability)·일관성(Conformity)·친숙함(Familiarity)·정책(Policy).
- 자동화(automation), 통합/일관성(consolidation), 전문성으로 코드베이스가 커져도 일정 인력으로 작업을 수행하게 만드는 것이 목표.

### Shifting Left (좌측 이동)
> 개발 워크플로 타임라인에서 문제를 더 일찍("왼쪽") 발견할수록 수정 비용이 싸진다.

- 보안·정적분석·코드리뷰로 커밋 전에 잡는 결함이 프로덕션 결함보다 훨씬 저렴하다(defense-in-depth).

---

## 3. Trade-offs and Costs (트레이드오프와 비용)
> 좋은 엔지니어링 의사결정은 측정/추정 가능한 비용을 근거로 트레이드오프를 합리적으로 평가하는 것이며, "내가 그랬으니까(because I said so)"는 나쁜 이유다.

- 비용은 금전·자원(CPU)·인력·거래(transaction)·기회(opportunity)·사회적(societal) 비용을 포괄한다.
- 측정 가능한 결정(CPU vs RAM 변환표)과 측정 어려운 결정(설계 나쁜 API의 비용, 사회적 영향)을 구분해 후자에 더 신중해야 한다.
- 예시(Markers): 1달러짜리 마커를 통제하기보다 자유로운 브레인스토밍을 최적화 — 인력 비용이 보통 한계 요인.
- 예시(Distributed Builds): 분산 빌드로 생산성을 크게 올렸으나, 의존성 비대화(Jevons Paradox)라는 예상치 못한 비용이 따라옴.
- 데이터 기반 문화의 숨은 이점: 실수를 인정하고 결정을 재검토할 수 있는 능력 — 시간이 지나면 데이터도 바뀐다.

---

## Summary (핵심 정리)
- 소프트웨어 엔지니어링 = 프로그래밍 + 시간·규모·트레이드오프 차원; 지속가능성은 "필요한 변화에 대응할 수 있는 능력"이다.
- Hyrum's Law와 확장 가능한 정책(Beyoncé/Churn Rule), Shifting Left가 장기 코드베이스 건강의 핵심 메커니즘이다.
- AI로 구현 리소스가 극도로 낮아진 시대일수록, 문제 설정과 장기 트레이드오프 판단이라는 엔지니어링 본질이 시니어의 경쟁력이 된다.
