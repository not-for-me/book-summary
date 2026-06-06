# 05. Enabling Tool Use and Planning in Agents

## 챕터 개요 (3줄 요약)

- 에이전트가 외부 도구(tool)와 함수 호출(function calling)을 활용해 내재적 한계를 넘어서는 방법을 다룬다.
- STRIPS, A*, GraphPlan, MCTS부터 LLM 기반 계획과 HTN까지 다양한 계획 알고리즘의 실용성을 비교한다.
- 도구 사용과 계획을 통합하고, CrewAI·AutoGen·LangGraph 같은 실제 프레임워크 예시로 구현을 보여준다.

---

## 1. Understanding the concept of tool use in agents

> 도구 사용은 LLM 에이전트가 외부 자원·계측을 활용해 자신의 기능과 의사결정을 보강하는 능력이다.

- 에이전트를 내부 지식(학습 데이터)에만 의존하는 고립된 개체에서 벗어나게 한다.
- 예: "날씨가 어때?" 질문에 실시간 날씨 API 도구가 없으면 모델은 꾸며내거나 모른다고 답한다.
- 도구 접근이 있으면 최신·사실 기반 정보를 가져와 정확히 응답한다.
- 함수 호출(function calling)은 LLM이 구조화된 인자로 외부 함수를 호출하게 한다.
- 도구는 에이전트의 한계를 전략적으로 극복하는 핵심 수단이다.

### Defining tools for agents

- **프레임워크 방식(docstring 사용)**: 함수의 docstring과 시그니처로 도구를 자동 정의한다.
- **직접 LLM 통합**: 도구 스키마를 직접 모델에 전달해 호출하게 한다.

### Types of tools / significance

- 검색, 계산, API 호출, 코드 실행 등 다양한 유형의 도구가 존재한다.
- 도구는 에이전트를 사실에 기반(grounded)하게 만들어 신뢰성을 높인다.

```
User Query --> [LLM Agent] --decides--> [Tool: Weather API] --> Real Data
                    ^------------- grounded response --------------|
```

---

## 2. Planning algorithms for agents

> 계획은 에이전트가 행동을 추론하고 목표 달성을 위한 행동 순서를 결정하는 근본 능력이다.

- 알고리즘은 입력을 받아 유한한 단계로 기대 출력을 내는 명확한 절차다.
- LLM 에이전트에서는 자연어, 불확실성, 거대한 상태 공간(state space) 처리의 실용성이 중요하다.
- 알고리즘마다 강점과 접근이 다르며 실용성 기준으로 분류된다.

### Less practical planning algorithms

- **STRIPS**: 전제·효과 기반 고전적 기호 계획으로 상태 공간이 크면 비효율적이다.
- **A\* planning**: 휴리스틱 기반 최적 경로 탐색이나 LLM 상태 공간에는 부적합하다.
- **GraphPlan**: 계획 그래프로 탐색하나 자연어 환경 대응이 약하다.
- **MCTS (Monte Carlo Tree Search)**: 샘플링 기반이나 계산 비용이 크다.

### Moderately practical – FF (Fast-Forward)

- 휴리스틱 기반으로 STRIPS류보다 빠르지만 여전히 정형 도메인에 치우친다.

### Most practical planning algorithms

- **LLM-based planning**: 자연어로 목표를 분해·추론하여 유연하고 실용적이다.
- **HTN (Hierarchical Task Network)**: 작업을 계층적 하위 작업으로 분해해 복잡성을 관리한다.

---

## 3. Integrating tool use and planning

> 진정으로 지능적인 에이전트는 도구 사용과 계획을 효과적으로 통합해 사실에 근거한 결과를 만든다.

- 기존 연구는 계획과 도구를 분리해 다뤘으나, 통합이 필수적이다.
- 예: 여행 플래너가 만든 계획이 허구였다면, 실제 항공·호텔 데이터를 도구로 주입해 사실 기반으로 만든다.
- **도구에 대한 추론(reasoning about tools)**: 각 도구의 기능·한계·맥락을 이해한다.
- **도구 사용을 위한 계획(planning for tool use)**: 어떤 도구를 언제 호출할지 순서를 계획한다.

```
[Goal] -> (Plan) -> step1: call Tool A -> step2: call Tool B -> [Grounded Result]
```

---

## 4. Exploring practical implementations

> CrewAI, AutoGen, LangGraph는 도구 사용과 계획을 결합한 에이전트를 구축하는 대표 프레임워크다.

- **CrewAI**: 역할 기반 에이전트들이 협업해 작업을 수행하는 프레임워크다.
- **AutoGen**: 다중 에이전트 대화·협력을 통해 작업을 해결한다.
- **LangGraph**: 그래프 기반으로 에이전트 워크플로우와 상태 흐름을 명시적으로 정의한다.
- 각 프레임워크는 도구 호출과 계획 통합을 서로 다른 추상화로 제공한다.

---

## Summary (핵심 정리)

- 도구 사용은 에이전트를 고립된 모델에서 벗어나 사실에 근거한 행동을 가능하게 하는 핵심 능력이다.
- 계획 알고리즘 중 LLM 기반 계획과 HTN이 자연어·불확실성·대규모 상태 공간에 가장 실용적이다.
- 도구와 계획의 통합이 정확한 작업 수행의 열쇠이며, CrewAI·AutoGen·LangGraph로 실제 구현할 수 있다.
