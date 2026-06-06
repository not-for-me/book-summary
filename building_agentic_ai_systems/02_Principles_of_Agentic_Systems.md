# 02. Principles of Agentic Systems

## 챕터 개요 (3줄 요약)

- 에이전트 시스템의 기본 원리로서 자기통치(self-governance), 행위성(agency), 자율성(autonomy)의 개념을 정의하고 구분한다.
- 지능형 에이전트의 특성과 더불어 숙고형(deliberative)·반응형(reactive)·하이브리드(hybrid) 아키텍처를 비교한다.
- 여행 예약 어시스턴트 예시를 통해 다중 에이전트 시스템(MAS, Multi-Agent System)의 구조와 상호작용 메커니즘을 설명한다.

---

## 1. Understanding self-governance, agency, and autonomy

> 에이전트 시스템의 핵심은 특정 맥락에서 선택을 최적화하는 정교한 의사결정 과정에 있으며, 이는 책임과 설명가능성에 대한 기존 관념에 도전한다.

- **자기통치(Self-governance)**: 외부의 직접 통제 없이 스스로 규칙과 목표에 따라 행동을 규율하는 능력이다.
- **행위성(Agency)**: 환경을 인식하고 목표 달성을 위해 능동적으로 행동을 선택·수행하는 능력이다.
- **자율성(Autonomy)**: 사람의 개입을 최소화하며 독립적으로 의사결정을 내리는 정도를 의미한다.
- 세 개념은 서로 연결되어 있으나, 자율성의 수준은 시스템마다 스펙트럼으로 존재한다.
- 로봇공학, AI 등 다양한 분야에서 혁신을 이끄는 원동력이 되며, 책임 소재(accountability) 문제를 제기한다.
- 여행 예약 어시스턴트 예시에서, 사용자의 목표를 받아 스스로 항공·호텔을 선택·예약하는 과정이 agency와 autonomy를 보여준다.

### Example of agency and autonomy in agents

- 에이전트가 단순 명령 실행을 넘어, 제약 조건 하에서 스스로 최적의 행동을 선택할 때 진정한 행위성이 발현된다.
- 자율성이 높을수록 효율적이지만, 그만큼 신뢰성과 통제 가능성에 대한 설계 고려가 중요해진다.

---

## 2. Reviewing intelligent agents and their characteristics

> 지능형 에이전트는 환경을 인식(perceive)하고, 추론(reason)하며, 학습(learn)하고, 행동(act)하는 능력을 갖춘 자율적 개체다.

- 지능형 에이전트는 센서로 환경을 인식하고 액추에이터(actuator)로 행동한다.
- 핵심 특성으로는 반응성(reactivity), 능동성(pro-activeness), 사회성(social ability), 자율성이 있다.
- 목표 지향적(goal-oriented)으로 동작하며 환경 변화에 적응한다.
- 학습 능력을 통해 경험으로부터 행동을 개선할 수 있다.
- 단일 에이전트의 한계는 다중 에이전트 협력으로 보완된다.

---

## 3. Exploring the architecture of agentic systems

> 에이전트 시스템은 숙고형·반응형·하이브리드라는 세 가지 주요 아키텍처 패턴으로 인식·추론·학습·행동을 구현한다.

- 아키텍처 패턴은 시스템이 환경을 인식하고 행동하는 구조와 동작 방식을 정의한다.
- 각 패턴은 응답 속도, 계획 능력, 복잡성 측면에서 트레이드오프를 가진다.
- 실제 시스템은 요구사항에 따라 패턴을 조합해 사용한다.

### Deliberative architectures

- 지식 기반(knowledge-based) 또는 기호적(symbolic) 아키텍처로도 불린다.
- 명시적 지식 표현과 추론 메커니즘에 기반해 sense-plan-act 주기를 따른다.
- 복잡한 추론이 필요한 작업에 강하지만 응답이 느릴 수 있다.

### Reactive architectures

- 내부 모델 없이 환경 자극에 즉각 반응하는 구조다.
- 빠르고 견고하지만 장기 계획 능력은 제한적이다.

### Hybrid architectures

- 숙고형의 계획 능력과 반응형의 즉각성을 결합한다.
- 일반적으로 계층적(layered) 구조로 구현되어 실용적 균형을 제공한다.

```
+-------------------- Hybrid Architecture --------------------+
| Deliberative Layer:  Goal / Planning / Knowledge Base       |
|        |  (plans)                                           |
| Reactive Layer:      Sense -> Immediate Action              |
+-------------------------------------------------------------+
        Environment  <----- perceive / act ----->
```

---

## 4. Understanding multi-agent systems

> MAS(Multi-Agent System)는 분산 인공지능의 한 분야로, 여러 자율 에이전트가 상호작용·협력·조정하여 집단 목표를 달성한다.

- MAS는 단일 에이전트가 풀기 어려운 복잡한 문제를 분산·협력으로 해결한다.
- 각 에이전트는 자율적으로 인식·추론·행동하며, 집단적 행동에서 창발적(emergent) 결과가 나타난다.
- 공급망 관리, 물류, 교통 제어 등 다양한 도메인에 적용된다.

### Definition and characteristics of MASs

- 분산성, 자율성, 상호작용성, 확장성(scalability)이 주요 특징이다.
- 중앙 통제가 없거나 약해도 협력을 통해 전체 목표를 달성할 수 있다.

### Interaction mechanisms in MASs

- 에이전트 간 통신은 협상(negotiation), 협력(cooperation), 조정(coordination) 메커니즘으로 이루어진다.
- 공통 프로토콜과 메시지 교환을 통해 충돌을 해소하고 목표를 정렬한다.

```
[Agent A] <--negotiate--> [Agent B]
     \                      /
      \---> [Shared Goal] <--/   (coordination & cooperation)
```

---

## Summary (핵심 정리)

- 에이전트 시스템은 self-governance, agency, autonomy라는 핵심 원리 위에서 동작하며 자율적 의사결정이 본질이다.
- 숙고형·반응형·하이브리드 아키텍처는 각각 계획성과 즉각성의 트레이드오프를 가지며, 하이브리드가 실용적 균형을 제공한다.
- MAS는 다수의 자율 에이전트가 협력·조정하여 단일 에이전트의 한계를 넘어 복잡한 문제를 해결한다.
