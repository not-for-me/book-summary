# 09. Software Architecture

## 챕터 개요 (3줄 요약)
- 아키텍처는 나중에 바꾸기 어려운 결정이며, "구글로 검색할 수 없는 것"이자 트레이드오프 분석의 결과다.
- 모든 질문의 답은 "It depends"이며, 품질 속성(quality attributes, illities)을 식별하고 올바른 긴장 관계로 배치하는 것이 핵심이다.
- 진화적 아키텍처, 적합성 함수(fitness functions), 다이어그램, ADR(Architecture Decision Records)로 아키텍처를 유지·소통한다.

---

## 1. What Is Architecture?
> 아키텍처는 규모·성능·보안 등 품질 속성을 고려한 시스템의 전체 설계이며, 나중에 바꾸기 어려운 결정들이다.

- "아키텍처"라는 용어는 건축 산업에서 빌려왔으며, 콘크리트를 부은 뒤엔 리팩터링이 어렵다.
- 아키텍트는 큰 그림, 구조, 비전 — 전기·배관처럼 나중에 바꾸기 힘든 것들을 책임진다.
- 데이터스토어 오류는 검색으로 답을 찾을 수 있지만, "Foo 앱에 어떤 데이터스토어를 쓸까?"는 검색으로 답할 수 없다.
- 아키텍트는 사용자 수, 위치, 가용성 목표, 제약, 규제, 응답 시간, 보안 정책 등 수많은 질문을 던진다.
- 모든 앱은 고유하므로 일반적 정답이 없으며, 모든 결정의 장단점을 따져야 한다.

---

## 2. Trade-Offs
> 아키텍처는 단순한 답이 없는 질문을 다루므로 거의 모든 질문에 "It depends"로 답한다.

- "아키텍처에 옳고 그른 답은 없고 트레이드오프만 있다"(Neal Ford) — 보통 "least worst answer"를 받아들인다.
- 아키텍트의 일은 여러 옵션에 대한 트레이드오프 분석이며, 별점이 가장 많은 것을 고르는 단순 작업이 아니다.
- "우연한 아키텍트(accidental architect)": 직함 없이 아키텍처 작업을 하게 될 수 있으니 큰 그림을 보고 피드백을 구한다.
- 예: 재사용 라이브러리를 런타임 의존성으로 할지 빌드타임 의존성으로 할지는 변동성·최신성 요구에 따라 다르다.
- 세금 계산 유틸 예: 자주 변하지만 네트워크 장애 시 백업이 필요하다 — 모든 결정은 트레이드오프다.

---

## 3. Architecture Versus Design
> 아키텍처와 설계의 경계는 모호하며(Twin Peaks Model), 대부분의 결정은 둘 사이 연속선상에 있다.

- 설계 결정이 아키텍처에 영향을 주고 그 반대도 성립한다(닭과 달걀).
- 전략적(strategic)일수록 아키텍처, 전술적(tactical)일수록 설계에 가깝다.
- 판단 기준: 의사결정에 필요한 인원/역할, 변경 비용(메서드 리팩터링=전술적, 모놀리식→마이크로서비스=전략적).
- 아키텍트와 개발자는 연속선 중간의 결정을 함께 다룰 준비가 되어야 한다.

---

## 4. Quality Attributes
> 아키텍트는 기능을 넘어 완전한 솔루션을 위한 품질 속성(비기능 요구사항, "illities")을 고려해야 한다.

- "비기능 요구사항"이라 하면 이해관계자가 듣기를 멈추므로 "품질(quality)"로 대화 톤을 바꾼다.
- 유지보수성, 확장성, 신뢰성, 보안, 배포성 등 — 어떤 것이 가장 중요한지는 "It depends".
- 모든 노브를 11까지 올릴 수 없으며, 일부 속성은 반비례 관계다(보안 vs 사용성).

### Identifying & Aligning
- 요구사항의 특정 단어/구절이 품질 속성을 신호한다(예: Concert 티켓팅의 동시 사용자 폭증 → 확장성·회복성).
- 품질 속성을 순위화(마인드맵/표/번호 목록)하고 이해관계자와 공유·반복한다.
- 이해관계자 정렬에는 영향력 기술이 필요하며, 기술 용어가 아닌 상대에게 관련된 예시로 설명한다.
- 간접의 힘(power of indirection): 직접 경로가 가장 큰 저항을 부르므로 대안적 프레이밍을 고려한다(Git을 대체가 아닌 보완으로 도입한 사례).

---

## 5. Architectural Styles
> 핵심 품질 속성을 식별한 뒤 적절한 아키텍처 스타일을 분석·선택하며, 만능 해법은 없다.

- 크게 모놀리식(단일 배포 단위)과 분산(다중 배포 단위)으로 나뉜다.
- 모놀리식은 불공정하게 비난받지만, 잘 구조화된 모듈러 모놀리스도 가능하다.
- 모놀리식 스타일: big ball of mud, layered, microkernel, pipeline.
- 분산 스타일: space based, service oriented, microservices, event driven.
- 마이크로서비스는 탄력성·확장성이 뛰어나지만 네트워크 홉, 복잡도, 비용, 디버깅 난이도가 증가한다.

---

## 6. The Agile Architect & Fitness Functions
> 아키텍처와 애자일은 공존할 수 있으며, 변화를 기대하고 설계하는 진화적 아키텍처가 핵심이다.

- 진화적 아키텍처(evolutionary architecture)는 여러 차원에서 안내되고 점진적인 변화를 지원한다.
- 가설 주도 개발(hypothesis-driven development): "We believe ~ Will result in ~ We will know we succeeded when ~".
- 데이터 기반 결정이 "가장 목소리 큰 사람"에 따르는 것보다 낫다.

### Fitness Functions
- 열역학 제2법칙처럼 노력 없이는 아키텍처가 무질서로 퇴화한다.
- 적합성 함수(fitness function)는 아키텍처가 목표에 얼마나 부합하는지 지속적으로 측정·테스트한다.
- 예: 서비스 호출 평균 100ms 이내, 순환 복잡도 5 초과 금지 — 가능한 한 CI/CD에서 자동화한다.

```
Evolutionary architecture loop:
  design for change -> deploy -> measure (fitness functions)
                          ^                     |
                          |_____ feedback ______|
```

---

## 7. Architectural Diagrams & ADRs
> 아키텍처는 결국 소통이며, 청중에 맞는 다이어그램과 결정의 "왜"를 기록하는 ADR로 표현한다.

### Diagrams
- 청중(개발자/아키텍트/비즈니스 파트너)을 먼저 고려하고, 스토리 전달에 도움이 될 때만 다이어그램을 만든다.

### Architectural Decision Records (ADRs)
- 소프트웨어 아키텍처 제2법칙: "왜(why)가 어떻게(how)보다 중요하다."
- ADR(Michael Nygard 도입)은 결정의 동기를 순차 로그로 기록하며 Markdown/AsciiDoc 평문이 흔하다.
- 구성: Title(3자리 ID), Status(RFC/Proposed/Accepted/Superseded), Context, Options, Decision, Consequences, Governance, Notes.
- ADR은 불변(immutable)이며, 나중 ADR이 이전 것을 대체(superseded)할 수 있다.
- 버전 관리에 저장하고 반드시 검색 가능하게 한다.

---

## Summary (핵심 정리)
- 아키텍처는 큰 그림을 보고 핵심 품질 속성을 식별하며 결정마다의 트레이드오프를 헤쳐나가는 거대한 주제다.
- 모든 질문의 답은 "It depends"이며, 학문적으로 완벽한 아키텍처가 아닌 마감 내 최선의 결정을 내린다.
- 진화적 아키텍처, 적합성 함수, 적절한 다이어그램, ADR로 아키텍처를 유지·소통하며 조직 정치를 항해한다.