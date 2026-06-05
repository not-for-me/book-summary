# 08. Reinforcement Learning and AI Agents

## 챕터 개요 (3줄 요약)

- 강화학습(RL)은 에이전트가 환경과 상호작용하며 시행착오로 누적 보상을 최대화하는 패러다임이다.
- 다중 슬롯 밴디트, 마르코프 결정 과정(MDP), 심층 RL 알고리즘(DQN, REINFORCE, PPO, 액터-크리틱, AlphaZero)을 다룬다.
- LLM과 RL의 상호작용(정보 처리기, 보상 설계자, 의사결정자, 생성기)을 살펴본다.

---

## 1. Introduction to reinforcement learning

> RL은 레이블 데이터 대신 보상으로부터 학습하며, 탐험(exploration)과 활용(exploitation)의 균형을 추구한다.

- RL 요소: 에이전트(agent), 환경(environment), 상태(state), 정책(policy), 보상(reward), 가치 함수(value function).
- 지도/비지도 학습과 달리 RL은 경험(trial and error)과 지연된 보상(delayed reward)으로 학습한다.
- 정책(policy)은 상태에 행동을 매핑하며 흔히 신경망으로 매개변수화된다.
- 보상은 즉각적 신호, 가치 함수는 장기적 기대 수익을 나타낸다.
- 에이전트는 상태 전체가 아닌 관측(observation, 부분 정보)만 접근할 수 있다.

### RL 상호작용 루프

```
state S_t -> [policy pi] -> action A_t -> environment
  -> reward R_t + next state S_(t+1) -> repeat
```

---

## 2. The multi-armed bandit problem

> k-armed bandit는 에이전트가 시행착오로 가장 보상이 큰 레버를 찾는 RL의 고전적 예제다.

- 각 레버(bandit)는 고유의 확률 분포를 가지며 에이전트는 추정 가치 Qt(a)를 학습한다.
- 그리디(greedy) 행동은 추정 가치가 가장 높은 행동을 선택(활용)한다.
- ε-greedy 방법은 확률 ε로 무작위 탐험을 해 활용과 탐험을 균형맞춘다.
- 순수 greedy(ε=0)는 최적 선택을 1/3만, ε=0.1은 80% 선택해 탐험의 이점을 보인다.
- 증분적(incremental) 업데이트로 평균을 매번 재계산하지 않고 고정 스텝 크기 α로 비정상 문제에 대응한다.
- 낙관적 greedy(optimistic)는 초기 값을 높게 설정해 탐험을 유도한다.
- UCB(Upper Confidence Bound)는 불확실성을 고려해 비그리디 행동을 선택한다.

---

## 3. Markov decision processes

> MDP는 행동이 즉각 보상뿐 아니라 미래 상태에도 영향을 주는 지연 보상 문제를 형식화한다.

- 상태 전이 확률 함수 p(s',r|s,a)로 시스템 동역학을 기술한다.
- 마르코프 상태(Markov state)는 미래 예측에 필요한 모든 정보를 담고 있다.
- 할인율(discount rate) γ로 즉각 보상과 미래 보상의 가중치를 조절한다.
- 상태-가치 함수 v_pi(s)와 행동-가치 함수 q_pi(s,a)로 정책을 평가한다.
- 벨만 방정식(Bellman equation)은 재귀적 형태로 가치를 계산하며 많은 RL 알고리즘의 기반이다.
- 최적 정책 pi*는 모든 상태에서 최대 가치 함수를 가진다.
- 동적 계획법(DP)은 완벽한 모델을 가정한 근사 방법이며 현대 알고리즘의 영감이 되었다.

---

## 4. Deep reinforcement learning

> 심층 RL은 신경망으로 정책과 가치 함수를 표현해 고차원 상태 공간 문제를 해결한다.

- 모델 프리(model-free): 환경 모델 없이 관측에서 직접 학습(구현 간단, 확장성 좋음).
- 모델 기반(model-based): 내부 환경 모델로 미래를 시뮬레이션해 계획하며 샘플 효율이 높다.
- 온폴리시(on-policy): 현재 정책의 데이터로 학습(안정적, 샘플 비효율).
- 오프폴리시(off-policy): 다른 정책 경험으로 학습(샘플 효율, 탐험 유연).
- Q-learning/DQN(Deep Q-Network)은 Q-테이블/신경망으로 가치를 학습하며 경험 재생(experience replay)과 타깃 네트워크로 안정화한다.
- REINFORCE는 확률적 정책을 직접 학습하는 정책 그래디언트 방법이다.
- PPO(Proximal Policy Optimization)는 클리핑(clipping)으로 정책 업데이트를 안정화한다.
- 액터-크리틱(actor-critic)은 정책(actor)과 가치 평가(critic)를 결합한다.
- AlphaZero는 심층 학습과 MCTS(Monte Carlo Tree Search)를 결합한 자가대국(self-play) 모델 기반 RL이다.
- 실습: A3C/A2C로 비디오게임 에이전트를 학습하며 PPO로 GPU 최적화 가능하다.

### DQN 구조

```
Q-network -> predicted Q-value
target network -> target Q-value (periodic copy)
experience replay buffer -> random batch -> stable training
```

---

## 5. LLM interactions with RL models

> LLM과 RL의 세 가지 상호작용: RL이 LLM 강화, LLM이 RL 강화, 둘을 결합하는 방식이 있다.

- RL이 LLM 강화: RLHF(Reinforcement Learning from Human Feedback)와 프롬프트 최적화에 PPO를 사용한다.
- 정보 처리기(information processor): LLM이 특징 추출기나 자연어를 형식 언어로 변환해 에이전트 학습을 돕는다.
- 보상 설계자(reward designer): LLM이 암시적/명시적 보상 모델로 보상 함수를 생성한다.
- 의사결정자(decision-maker): LLM이 행동 공간을 줄여 탐험 효율을 높이고 시퀀스 모델링으로 결정한다.
- 생성기(generator): LLM이 세계 모델 시뮬레이터로 궤적을 생성하거나 XRL(Explainable RL)로 정책을 설명한다.
- 응용: 로보틱스, 자율주행, 의료 추천, 에너지 관리.
- 한계: LLM의 편향·환각 상속, 높은 연산 비용, 윤리·법적 문제.

### LLM-RL 결합 세 가지

```
RL -> enhance LLM   (RLHF, prompt optimization)
LLM -> enhance RL   (info processor / reward / decision / generator)
RL + LLM            (combined planning, no mutual training)
```

---

## Summary (핵심 정리)

- 에이전트가 환경을 탐험·수정하며 피드백으로 학습하는 능동적 RL 패러다임을 배웠다.
- 탐험과 활용의 균형, MDP, 심층 RL 알고리즘(DQN, REINFORCE, PPO, 액터-크리틱, AlphaZero)을 익혔다.
- LLM과 RL의 시너지를 이해했으며, 다음 장부터는 LLM 에이전트가 도구를 선택해 작업을 수행하는 데 집중한다.
