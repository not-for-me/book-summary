# 03. Essential Components of Intelligent Agents

## 챕터 개요 (3줄 요약)

- 지능형 에이전트를 구성하는 핵심 요소인 지식 표현(knowledge representation), 추론(reasoning), 학습(learning), 의사결정·계획(decision-making & planning)을 다룬다.
- 인간의 골격에 비유하여, 이 구성 요소들이 에이전트가 환경에 적응하고 목표를 추구하도록 뼈대를 제공함을 설명한다.
- 생성형 AI(Generative AI)가 이러한 구성 요소를 강화하여 더 강력한 이해·학습·상호작용 능력을 부여하는 방식을 소개한다.

---

## 1. Knowledge representation in intelligent agents

> 지식 표현은 에이전트가 환경에 대한 이해를 추론과 의사결정에 적합한 형식으로 인코딩하는 메커니즘이다.

- 정보를 구조화·조직화하여 에이전트가 추론, 문제 해결, 행동 선택에 활용하도록 만든다.
- 환경 모델(model of surroundings)을 구축하는 가장 기본적인 수단이다.
- 대표 방식으로 의미망(semantic network), 프레임(frame), 논리 기반 표현(logic-based representation)이 있다.
- 각 방식은 표현력, 추론 효율, 적용 영역 측면에서 강점이 다르다.
- 표현 형식의 선택은 이후 추론과 학습의 성능에 직접 영향을 준다.

### Semantic networks

- 노드(개념)와 엣지(관계)로 지식을 그래프 형태로 표현하는 직관적 방식이다.
- 개념 간 연관과 상속(inheritance) 관계를 자연스럽게 모델링한다.

### Frames

- 객체의 속성과 기본값을 슬롯(slot) 형태로 묶어 구조적으로 표현한다.
- 정형화된 상황·객체 표현에 적합하다.

### Logic-based representations

- 명제·술어 논리를 사용해 엄밀한 추론을 지원한다.
- 형식적 정확성이 높지만 표현과 계산 비용이 클 수 있다.

```
[Bird] --is_a--> [Animal]
   |--has--> [Wings]
   |--can--> [Fly]
```

---

## 2. Reasoning in intelligent agents

> 정교한 에이전트는 단일 추론이 아니라 여러 추론 방식과 데이터 기반·학습 요소를 결합한 다면적 추론을 사용한다.

- 질의응답 시스템은 의미 파싱, 연역 추론, 신경망 생성을 조합해 답을 만든다.
- 기본 추론 패러다임은 연역(deductive), 귀납(inductive), 가추(abductive)다.
- 연역은 일반 규칙에서 특정 결론을 도출하는 하향식(top-down) 추론이다.
- 귀납은 구체적 사례에서 일반 규칙을 추론하는 상향식 방식이다.
- 가추는 관찰된 결과를 가장 잘 설명하는 가설을 추론한다.

### Deductive / Inductive / Abductive

- **Deductive**: "모든 사람은 죽는다 → 소크라테스는 죽는다" 같은 삼단논법(syllogism)으로 결론이 논리적으로 필연적이다.
- **Inductive**: 다수 관찰로부터 일반화하나 결론이 확률적이다.
- **Abductive**: 불완전한 정보 하에서 가장 그럴듯한 설명을 선택한다.

---

## 3. Learning mechanisms for adaptive agents

> 학습 메커니즘은 에이전트가 경험으로부터 행동을 개선하고 변화하는 환경에 적응하도록 한다.

- 지도학습(supervised), 비지도학습(unsupervised), 강화학습(RL, Reinforcement Learning) 등 다양한 패러다임을 활용한다.
- 강화학습은 보상 신호를 통해 시행착오로 정책(policy)을 학습한다.
- 학습은 정적인 규칙 기반 시스템을 적응형 에이전트로 진화시킨다.
- 경험 데이터의 품질과 양이 학습 성능을 좌우한다.
- 생성형 AI는 적은 데이터로도 일반화를 돕는 사전학습(pretraining) 능력을 제공한다.

---

## 4. Decision-making and planning in agentic systems

> 의사결정과 계획은 에이전트가 목표 달성을 위해 효용을 최대화하는 행동 순서를 선택하는 과정이다.

- **효용 함수(Utility function)**: 가능한 행동·상태의 가치를 수치화해 최적 선택의 기준을 제공한다.
- **그래프 기반 계획(Graph-based planning)**: 상태 공간을 그래프로 탐색해 목표 경로를 찾는다.
- **휴리스틱 탐색(Heuristic search)**: A* 등 추정 비용으로 탐색을 효율화한다.
- **몬테카를로 트리 탐색(MCTS, Monte Carlo Tree Search)**: 시뮬레이션 기반 샘플링으로 거대한 탐색 공간을 다룬다.
- **계층적 계획(Hierarchical planning)**: 큰 목표를 하위 목표로 분해해 복잡성을 관리한다.
- **제약 만족(Constraint satisfaction)**: 제약 조건을 만족하는 해를 탐색한다.

### Planning algorithms

- 알고리즘 선택은 문제의 규모, 불확실성, 실시간 요구에 따라 달라진다.
- 휴리스틱과 계층적 분해는 대규모 문제의 계산 부담을 줄이는 핵심 전략이다.

```
        [Goal]
          |  hierarchical decomposition
   +------+------+
[Subgoal A]  [Subgoal B]
   |              |
[Actions]      [Actions]   --> Utility-based selection
```

---

## 5. Enhancing agent capabilities with generative AI

> 생성형 AI는 지식 표현·추론·학습·계획 전반을 강화하여 에이전트의 이해와 상호작용 능력을 크게 끌어올린다.

- LLM(Large Language Model)은 비정형 지식을 유연하게 표현하고 자연어 추론을 가능하게 한다.
- 생성 능력으로 합성 데이터·시나리오를 만들어 학습과 계획을 보강한다.
- 이는 본격적인 agentic AI 구축의 출발점이 된다.

---

## Summary (핵심 정리)

- 지능형 에이전트의 핵심 구성 요소는 지식 표현, 추론, 학습, 의사결정·계획이며 각각이 에이전트의 "뼈대"를 이룬다.
- 추론은 연역·귀납·가추를 조합하고, 계획은 효용 함수와 다양한 탐색 알고리즘으로 최적 행동을 선택한다.
- 생성형 AI는 이 모든 구성 요소를 강화하여 더 적응적이고 강력한 에이전트를 가능하게 한다.
