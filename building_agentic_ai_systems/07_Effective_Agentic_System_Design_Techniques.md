# 07. Effective Agentic System Design Techniques

## 챕터 개요 (3줄 요약)

- 효과적인 에이전트 시스템 설계 기법으로 집중된 시스템 프롬프트, 상태 공간·환경 모델링, 메모리 아키텍처, 순차·병렬 처리를 다룬다.
- 명확한 목표·작업 명세·맥락 인식이 에이전트 성능을 좌우한다는 점을 강조한다.
- 단기·장기·일화(episodic) 메모리 구조와 워크플로우 최적화를 여행 에이전트 예시로 설명한다.

---

## 1. Focused system prompts and instructions for agents

> 집중된 지시는 에이전트의 목표, 제약, 운영 맥락을 규정하여 성능에 결정적 영향을 미친다.

- 명령의 명확성과 명시성이 목표 달성 성능을 크게 좌우한다.
- **목표 정의(defining objectives)**: 에이전트의 기능과 행동의 기반이 되는 명확한 목표를 설정한다.
- 예: 여행 에이전트의 목표는 개인화된 솔루션으로 고객 만족을 극대화하는 것이다.
- **작업 명세(task specifications)**: 수행할 구체적 작업과 기준을 정의한다.
- **맥락 인식(contextual awareness)**: 현재 상황과 사용자 맥락을 반영해 행동을 조정한다.

---

## 2. State spaces and environment modeling

> 상태 공간과 환경 모델링은 에이전트가 운영 맥락을 인식·이해·상호작용하는 토대를 형성한다.

- **상태 공간 표현(state space representation)**: 현재 상황·가용 행동·잠재 결과에 대한 이해를 유지·갱신한다.
- 잘 설계된 상태 공간은 불필요한 복잡성 없이 관련 정보를 추적한다.
- 여행 에이전트 예: 고객 프로필 상태(선호·이력·예산)와 여행 맥락 상태(항공·호텔 옵션)를 포함한다.
- **환경 모델링(environment modeling)**: 외부 환경의 동역학을 표현한다.
- **통합·상호작용 패턴**과 **모니터링·적응(monitoring & adaptation)**으로 일관된 행동을 유지한다.

```
[Environment] --perceive--> [State Space]
                               |
                          [Decision] --act--> [Environment]
                               ^---- monitor & adapt ----|
```

---

## 3. Agent memory architecture and context management

> 메모리 아키텍처와 맥락 관리는 과거 경험과 현재 맥락에 기반한 일관된 상호작용과 의사결정을 가능하게 한다.

- 에이전트 메모리는 일반적으로 단기·장기·일화 세 유형으로 구성된다.
- 각 메모리는 서로 다른 목적을 수행하며 의사결정에 통합된다.
- 맥락 관리(context management)는 한정된 컨텍스트 윈도우를 효율적으로 활용한다.

### Short-term memory (working memory)

- 현재 상호작용·작업에 관련된 정보를 일시적으로 보관하는 즉각적 인지 작업공간이다.
- 대화의 일관성 유지에 핵심적이다.

### Long-term memory (knowledge base)

- 지속적 지식과 사실을 저장하는 지식 기반(knowledge base)이다.
- 도메인 지식과 학습된 패턴을 장기 보관한다.

### Episodic memory (interaction history)

- 과거 상호작용 이력을 사건 단위로 저장한다.
- 개인화와 경험 기반 적응을 지원한다.

```
+------------ Agent Memory ------------+
| Short-term (working) : current task  |
| Long-term (knowledge): facts/domain  |
| Episodic (history)   : past episodes |
+--------------------------------------+
        --> integrated into Decision-making
```

---

## 4. Sequential and parallel processing in agentic workflows

> 에이전트 워크플로우는 순차 처리와 병렬 처리를 적절히 조합해 효율을 최적화한다.

- **순차 처리(sequential processing)**: 작업 간 의존성이 있을 때 단계별로 순서대로 실행한다.
- **병렬 처리(parallel processing)**: 독립적인 작업을 동시에 실행해 처리량을 높인다.
- **워크플로우 최적화(workflow optimization)**: 의존성 분석으로 병렬화 가능 작업을 식별한다.
- 잘못된 병렬화는 경쟁 조건(race condition)을 유발하므로 의존성 관리가 중요하다.
- 순차·병렬의 균형이 지연(latency)과 정확성 사이의 트레이드오프를 결정한다.

```
Sequential: A -> B -> C
Parallel:   A --+--> B --+--> Merge
                +--> C --+
```

---

## Summary (핵심 정리)

- 효과적인 에이전트 설계는 명확한 시스템 프롬프트와 목표·작업 명세, 그리고 맥락 인식에서 출발한다.
- 상태 공간·환경 모델링과 단기·장기·일화 메모리 아키텍처가 일관된 의사결정의 기반이 된다.
- 순차·병렬 처리를 의존성에 맞게 조합하여 워크플로우를 최적화하는 것이 성능의 열쇠다.
