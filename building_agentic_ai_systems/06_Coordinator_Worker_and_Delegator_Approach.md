# 06. Exploring the Coordinator, Worker, and Delegator Approach

## 챕터 개요 (3줄 요약)

- CWD(Coordinator-Worker-Delegator) 모델은 협업·전문화·작업 분배를 강조하는 다중 에이전트 시스템 설계 프레임워크다.
- 조직심리학과 경영 이론에서 영감을 받아, 역할 분담과 계층 구조를 지능형 에이전트에 적용한다.
- 여행 계획 에이전트 예시를 통해 역할 할당, 통신·협업 메커니즘, 생성형 AI에서의 구현 방법을 설명한다.

---

## 1. Understanding the CWD model

> CWD 모델은 협업·전문화·효과적 작업 및 자원 분배를 촉진하기 위해 설계된 다중 에이전트 프레임워크다.

- 인간 조직의 명확한 역할 위임과 계층 구조의 이점을 에이전트에 적용한다.
- 조직심리학·경영 이론의 검증된 협력 원리를 지능형 에이전트 분야에 이식한다.
- 시스템이 복잡해지고 여러 전문 능력이 협력해야 할 때 특히 유용하다.
- 단일 에이전트로는 불가능한 복잡한 목표를 자율 에이전트들의 협업으로 달성한다.
- 세 가지 역할(조정자·작업자·위임자)로 책임을 분리한다.

### Key principles / Travel agent example

- **Coordinator(조정자)**: 전체 작업을 분해하고 일정·정합성을 관리하는 전략적 총괄자.
- **Worker(작업자)**: 특정 전문 작업(예: 항공·호텔 검색)을 실행하는 전문가.
- **Delegator(위임자)**: 작업을 적절한 작업자에게 배분·위임한다.
- 여행 계획 시스템에서 각 역할이 협력해 고객 요구를 충족한다.

```
            [Coordinator]
          (plan & oversee)
                |
           [Delegator]  (assign tasks)
          /     |      \
   [Worker1] [Worker2] [Worker3]
   (flights) (hotels)  (activities)
```

---

## 2. Designing agents with role assignments

> 각 에이전트는 명확히 정의된 역할과 책임을 부여받아 시스템 목표에 기여한다.

- **여행 계획 에이전트(coordinator)**: 고객 요청을 관리 가능한 단위로 분해하고 일정과 기대치를 정렬한다.
- 전체 계획의 통합적 관점(holistic view)을 유지하며 우발 상황에 대응한다.
- 각 작업자 에이전트는 자신의 전문 영역에 집중해 효율을 높인다.
- 위임자는 작업과 작업자 능력을 매칭한다.
- 역할 분리는 책임 소재를 명확히 하고 시스템 확장성을 높인다.

---

## 3. Communication and collaboration between agents

> CWD 기반 시스템에서 에이전트 간 효과적인 통신과 협업은 성공적 결과 달성의 핵심이다.

- 에이전트는 정보 공유, 행동 조정, 협력적 행동을 수행할 수 있어야 한다.

### Communication / Coordination / Negotiation / Knowledge sharing

- **통신(Communication)**: 메시지 형식과 상호작용 패턴을 정의한 프로토콜을 따른다.
- **조정 메커니즘(Coordination)**: 작업 순서와 의존성을 맞춰 충돌 없이 진행한다.
- **협상·충돌 해소(Negotiation & conflict resolution)**: 자원·우선순위 충돌을 해결한다.
- **지식 공유(Knowledge sharing)**: 상태와 정보를 공유해 일관된 의사결정을 한다.

---

## 4. Implementing the CWD approach in generative AI systems

> 생성형 AI에서 CWD는 시스템 프롬프트, 명령 포맷팅, 상호작용 패턴을 통해 구현된다.

- **시스템 프롬프트(system prompts)**: 각 에이전트의 역할·행동을 정의한다.
- **명령 포맷팅(instruction formatting)**: 일관된 형식으로 작업 지시를 전달한다.
- **상호작용 패턴(interaction patterns)**: 에이전트 간 메시지 흐름과 호출 구조를 설계한다.
- LLM(Large Language Model)을 각 역할에 맞게 프롬프트하여 전문화된 행동을 유도한다.

---

## Summary (핵심 정리)

- CWD 모델은 조정자·작업자·위임자 역할 분담으로 복잡한 작업을 협력적으로 해결하는 다중 에이전트 프레임워크다.
- 명확한 역할 할당과 통신·조정·협상·지식 공유 메커니즘이 성공적 협업의 핵심이다.
- 생성형 AI에서는 시스템 프롬프트와 상호작용 패턴 설계를 통해 CWD를 실질적으로 구현한다.
