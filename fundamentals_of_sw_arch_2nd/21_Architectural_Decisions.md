# 21. Architectural Decisions

## 챕터 개요 (3줄 요약)

- 아키텍처 결정은 충분한 정보 수집·정당화·문서화·소통으로 개발팀을 올바른 기술 선택으로 안내한다.
- 세 가지 결정 안티패턴(Covering Your Assets, Groundhog Day, Email-Driven)을 극복해야 한다.
- ADR(Architectural Decision Records)로 결정을 문서화하며, 생성형 AI는 지혜가 부족해 결정 자체엔 한계가 있다.

---

## 1. Architectural Decision Antipatterns (결정 안티패턴)

> 세 안티패턴은 순차적 흐름을 이루며, 모두 극복해야 효과적 결정을 내릴 수 있다.

- Covering Your Assets(자산 보호): 틀릴까 두려워 결정을 회피/지연. 극복: '최후의 책임 있는 순간(last responsible moment)' — 지연 비용이 결정 위험을 초과하는 지점. 개발팀과 협업으로 검증(Analysis Paralysis 회피).
- Groundhog Day(성촉절): 결정 이유를 몰라 같은 논의 반복. 극복: 기술적·비즈니스적 정당화 모두 제공(비용·출시 기간·사용자 만족·전략적 위치).
- Email-Driven Architecture(이메일 주도): 결정을 잊거나 모름. 극복: 이메일 본문에 결정을 넣지 말고, 단일 진실 원천(위키·문서) 링크 제공. 영향받는 사람만 통지.

```
   cost vs risk over time:
   cost  ___/        last responsible moment
        /            = where cost rise > risk reduction
   risk \___
```

---

## 2. Architectural Significance (아키텍처적 중요성)

> Michael Nygard의 '아키텍처적으로 유의미한' 결정 — 구조·비기능 특성·의존성·인터페이스·구축 기법에 영향.

기술 결정도 아키텍처 특성(성능·확장성)을 직접 지원하면 아키텍처 결정이다. 구조(패턴·스타일), 비기능 특성, 의존성(결합점), 인터페이스(게이트웨이·계약·버저닝), 구축 기법(플랫폼·프레임워크·프로세스).

---

## 3. Architectural Decision Records (ADR)

> Michael Nygard가 전파한, 특정 결정을 기술하는 짧은 텍스트 파일(1~2쪽, Markdown/AsciiDoc).

### Basic Structure (기본 구조)

Title(번호+짧은 구문), Status(Proposed/Accepted/Superseded), Context(결정을 강제하는 상황·대안), Decision(결정+정당화, 명령형 voice), Consequences(영향·트레이드오프). 추가 권장: Compliance(거버넌스 방법 — 수동/적합성 함수), Notes(메타데이터).

- Status: Superseded는 이력 보존(ADR 42 ↔ ADR 68 상호 참조). RFC(Request for Comments) 상태로 피드백 수집 가능. 승인 기준(비용·교차팀 영향·보안)을 상사와 논의.
- Decision: "왜"가 "어떻게"보다 중요(gRPC 선택 이유를 모르면 후임이 REST로 바꿔 지연 발생).
- Compliance: ArchUnit/NetArchTest 적합성 함수로 측정 가능.

```
  ADR sections:
   Title | Status | Context | Decision | Consequences | (Compliance) | (Notes)
```

### Storing ADRs (저장)

각 결정은 별도 파일/위키 페이지. 대형 조직은 소스 Git이 아닌 전용 ADR 저장소·위키·공유 디렉터리 권장. 구조: common(전체 적용), application(앱별), integration(통신), enterprise(전사 글로벌).

### ADRs as Documentation / Standards / Existing Systems

ADR은 최선의 아키텍처 문서(C4 Model·ArchiMate는 다이어그램 표준). 표준에 ADR을 쓰면 "왜 존재하는가"를 정당화 — 정당화 못 하면 좋은 표준이 아님. 기존 시스템에도 유용(중요 결정의 'why' 발굴, 부적절한 설계 식별).

---

## 4. Generative AI and LLMs (생성형 AI 활용)

> LLM(Large Language Model)은 확률·'베스트 프랙티스' 기반이라 트레이드오프 분석이 필요한 결정엔 부적합하다.

아키텍처 결정은 비즈니스 관심사(출시 기간·성장)를 아키텍처 특성(유지보수성·테스트성 등)으로 번역하고 특정 맥락에서 트레이드오프 분석을 요구한다(예: 결제를 단일 서비스=성능 vs 유형별 서비스=유지보수성). 현재 생성형 AI의 최선은 트레이드오프를 나열해 놓친 부분을 돕는 것 — 지식은 있으나 지혜가 부족하다.

---

## Summary (핵심 정리)

- 효과적 결정은 세 안티패턴(회피·반복 논의·소통 부재)을 극복하고 비즈니스 가치를 정당화한다.
- 아키텍처적으로 유의미한 결정은 구조·비기능·의존성·인터페이스·구축 기법에 영향을 준다.
- ADR로 Context·Decision('왜')·Consequences를 문서화하며, 생성형 AI는 보조 도구로만 활용한다.
