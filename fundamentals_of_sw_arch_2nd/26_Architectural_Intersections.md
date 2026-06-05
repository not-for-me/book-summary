# 26. Architectural Intersections

## 챕터 개요 (3줄 요약)

- 아키텍처가 작동하려면 기술·비즈니스 환경의 다른 측면들과 정렬(intersection)되어야 한다.
- 구현·인프라·데이터 토폴로지·엔지니어링·팀·시스템 통합·엔터프라이즈·비즈니스·생성형 AI 9가지 교차점을 다룬다.
- 정렬 실패는 아무리 좋은 아키텍처도 목표 달성을 막으므로 소통·협업으로 정렬해야 한다.

---

## 1. Architecture and Implementation (구현)

> 소스 코드가 운영 관심사·내부 구조·제약과 정렬되어야 한다.

- Operational Concerns(운영): 아키텍처와 구현이 다른 목표를 좇으면 어긋남(microservices 확장성 목표 vs 인메모리 복제 캐시로 응답성 개선 → 8만 동시 사용자에서 OOM).
- Structural Integrity(구조 무결성): 소스 디렉터리 구조가 논리 아키텍처와 일치해야 함. ArchUnit/NetArchTest/PyTestArch/TSArch로 거버넌스.
- Architectural Constraints(제약): 구현이 제약을 어기면 실패(계층형에서 UI가 DB 직접 호출 → DB 변경이 전 계층 영향).

---

## 2. Architecture and Infrastructure (인프라)

> 인프라와 배포 방식이 확장성·응답성·내결함성·가용성 같은 운영 관심사와 정렬되어야 한다.

확장 가능한 아키텍처라도 인프라가 받쳐주지 않으면 무용(Pets.com 사례 — 마스코트에 돈 쓰고 인프라 부실로 탄력 확장 실패 → 폐업). 보통 아키텍트-운영 간 소통 부족이 원인이며 DevOps가 등장. microservices 창시자들은 운영 관심사를 운영이 맡게 해 설계를 단순화. 클라우드도 리전/존 분산이 캐시 이점을 상쇄하거나, co-location이 성능↑·확장성↓ 등 어긋날 수 있음.

---

## 3. Architecture and Data Topologies (데이터 토폴로지)

> 데이터 토폴로지·타입이 아키텍처 스타일과 정렬되어야 한다(자주 간과됨).

- Database Topology: 모놀리식/도메인/database-per-service. microservices는 database-per-service로 경계 컨텍스트 유지(service-based는 더 유연).
- Architectural Characteristics: DB 타입의 강점을 아키텍처 강점과 맞춤(확장성·탄력성 → key-value·columnar DB).
- Data Structure: 관계형 데이터→관계형 DB, 키-값/문서→해당 DB(polyglot 권장).
- Read/Write Priority: 쓰기 많음→columnar, 읽기 많음→key-value/document/graph, 균등→relational/NewSQL.

---

## 4. Architecture and Engineering Practices (엔지니어링 실천)

> 팀이 SW를 만들고 테스트·배포하는 방식이 아키텍처와 일치해야 한다.

프로세스(팀 구성·회의·워크플로우)와 엔지니어링 실천(프로세스 비종속 기법: XP, CI, CD, TDD)을 구분. SW의 약점은 추정(estimation). 반복적 프로세스가 아키텍처에 더 맞음(Waterfall로 microservices는 마찰). microservices는 자동 프로비저닝·테스트·배포 가정. Building Evolutionary Architectures의 적합성 함수로 진화·정렬. 시간 대 시장→민첩성(유지보수성·테스트성·배포성)을 적합성 함수로 추적.

---

## 5. Team Topologies / Systems Integration / Enterprise

> 팀 조직·시스템 통합·엔터프라이즈 표준과 정렬되어야 한다.

- Team Topologies: 도메인 분할 팀(크로스펑셔널)과 기술 분할 팀(UI·백엔드·DB 팀 → layered, 비즈니스·데이터 동기화 팀 → space-based)이 아키텍처와 맞아야 함.
- Systems Integration: 시스템은 고립되지 않음. 통신 프로토콜·계약·특성 호환성·quantum 보존을 고려(소홀하면 확장·응답성·민첩성 저하).
- Enterprise: 보안·플랫폼·문서·다이어그램 등 전사 표준과 정렬(무시하면 'one-off'로 폐기).

---

## 6. Architecture and Business Environment (비즈니스 환경)

> 비즈니스 환경과 도메인이 아키텍처에 직접 영향을 준다(domain-to-architecture isomorphism).

비용 절감 중인 회사는 고비용 microservices/space-based 부적합, 공격적 M&A 회사는 모놀리식 부적합. 'unknown unknowns'(아무도 모르는 변화)가 SW의 천적 — Big Design Up Front가 실패하는 이유. "모든 아키텍처는 unknown unknowns 때문에 반복적이며, Agile은 이를 일찍 인정할 뿐"(Mark). Barry O'Reilly의 residuality theory(변화=stressor, 대응=residue)로 임계 상태 도달.

---

## 7. Architecture and Generative AI (생성형 AI)

> LLM(Large Language Model) 통합이 아키텍처에 점점 중요한 교차점이 된다.

### 아키텍처에 Gen AI 통합

추상화·모듈성으로 LLM을 빠르게 교체하고, guardrails(rails)와 결과 평가(evals)를 가능하게 함(이력서 익명화 예시 — Langfuse로 관찰성).

### 아키텍트 보조로서의 Gen AI

특정 결정적 문제(코드 생성)엔 유용하나, "microservices vs space-based?" 같은 트레이드오프 결정엔 거의 정답 못 냄 — LLM은 지식은 있으나 지혜가 부족. 유망 도구: Thoughtworks Haiven(다이어그램 해석), PlantUML→ArchUnit 변환.

---

## Summary (핵심 정리)

- 아키텍처는 스타일 선택을 넘어 9가지 교차점(구현·인프라·데이터·엔지니어링·팀·통합·엔터프라이즈·비즈니스·AI)과 정렬되어야 한다.
- 정렬 실패는 좋은 아키텍처도 목표 달성을 막으므로 거버넌스 도구와 소통·협업이 필요하다.
- unknown unknowns 때문에 아키텍처는 본질적으로 반복적·진화적이어야 한다.
