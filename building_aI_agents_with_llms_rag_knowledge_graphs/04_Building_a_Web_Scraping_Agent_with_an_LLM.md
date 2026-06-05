# 04. Building a Web Scraping Agent with an LLM

## 챕터 개요 (3줄 요약)

- AI 에이전트는 LLM을 두뇌(brain)·지각(perception)·행동(action) 패러다임으로 확장한 자율 시스템이다.
- 에이전트 분류, 단일/다중 에이전트 능력, 작업 계획 방법과 주요 프레임워크(LangChain 등)를 다룬다.
- LangChain과 검색 도구(DuckDuckGo, Serper, Wikipedia)로 웹 검색 에이전트를 만드는 실습을 제공한다.

---

## 1. Understanding the brain, perception, and action paradigm

> AI 에이전트는 환경을 지각하고 결정을 내려 행동하는 인공 개체이며, 두뇌·지각·행동 세 부분으로 구성된다.

- LLM은 행동할 수 있으나 욕구·의도가 없어 의식적 존재가 아니며, 의식이라 부르는 것은 의인화 오류다.
- 에이전트의 네 가지 핵심 속성: 자율성(autonomy), 반응성(reactivity), 능동성(pro-activeness), 사회성(social ability).
- 두뇌(brain)는 LLM으로, 정보 저장·지식 탐색·추론·의사결정을 담당하며 자연어로 상호작용한다.
- 모델 지식은 언어적 지식, 상식(common-sense), 도메인 지식으로 분류된다.
- LLM 지식은 사전 학습 시점에 고정되어 지속 학습(continual learning)이나 과거 대화 기억이 불가하다.
- 지각(perception)은 이미지 캡셔닝, PaLM-E·BLIP-2(Q-Former) 같은 임베디드 모델로 시각·청각을 통합한다.
- 행동(action)은 도구(tool) 사용으로 LLM 능력을 확장하며, 외부 세계와 상호작용하는 체화(embodied) 행동도 포함한다.

### 에이전트 3요소 프레임워크

```
[ Perception ] --info--> [ Brain (LLM) ] --decision--> [ Action ]
   text/image/audio        reason/plan/memory          tools/embodiment
        ^----------------- environment feedback -----------------|
```

---

## 2. Classifying AI agents

> 에이전트는 가상 환경에 한정된 디지털 에이전트와 외부 세계와 상호작용하는 체화 에이전트로 나뉜다.

- 디지털 에이전트는 액션 에이전트(가상 세계 행동)와 인터랙티브 에이전트(세계 수정)로 확장된다.
- 작업 분해 방법: 분해 우선(decomposition-first)과 교차 분해(interleaved decomposition).
- 분해 우선은 전체 개요를 줘 환각을 줄이나 중간 오류 수정이 어렵다.
- 교차 분해는 동적 조정이 가능하나 복잡한 문제에서 긴 추론-계획 체인이 비싸진다.
- 다중 계획 선택(multi-plan selection)은 여러 후보 계획을 생성해 다수결·트리 탐색으로 선택한다.
- 외부 플래너 지원(external planner-aided), 반성·정제(reflection and refinement) 방법도 있다.
- 메모리 증강 계획(memory-augmented planning)과 RAG(Retrieval-Augmented Generation)로 컨텍스트 한계를 극복한다.

---

## 3. Understanding the abilities of single-agent and multiple-agent systems

> 단일 에이전트는 작업을 분해·계획·실행하며, 다중 에이전트는 협력·경쟁으로 복잡한 작업을 해결한다.

- 작업 지향 배포(task-oriented)는 웹 시나리오와 실제 시나리오(상식 추론 필요)로 나뉜다.
- 혁신 지향 배포(innovation-oriented)는 과학 탐구 같은 미래 응용이며, 생애주기 지향(life-cycle)은 스스로 탐험·학습한다.
- Minecraft는 단기·장기 작업 테스트베드로 활용된다.
- MAS(Multi-Agent System)에서 여러 LLM 에이전트가 자연어로 협력·통신하며 집단 의사결정을 내린다.
- 게임 이론에 따라 경쟁(adversarial setting)도 유익하며, AlphaGo는 자기 대국으로 학습했다.
- 인간-에이전트 상호작용은 불평등(instructor-executor)과 평등(equal) 패러다임으로 구분된다.
- 지속 학습(continual learning)은 매 상호작용마다 모델이 학습하는 하위 분야다.

---

## 4. Exploring the principal libraries

> LLM 기반 애플리케이션은 인터페이스, 두뇌(LLM), 지각 모듈, 도구, 프롬프트로 구성되며 여러 프레임워크로 구현한다.

- 프롬프트는 사용자에게 보이는 프론트엔드와 모델 행동을 조건화하는 백엔드 프롬프트로 나뉜다.
- LangChain은 가장 널리 쓰이며 LangChain·LangSmith·LangServe로 구성되고 체인(chain)을 만든다.
- LangChain은 풍부한 통합과 명확한 워크플로우가 장점이나 학습 곡선이 가파르고 문서가 부실하다.
- Haystack은 컴포넌트·파이프라인 구조로 RAG·Q&A에 강하나 사용자 기반과 확장성이 약하다.
- LlamaIndex는 160개 이상 데이터 소스를 다루며 인덱싱·검색이 강점인 RAG 1순위 선택이다.
- Semantic Kernel은 Microsoft의 함수 합성(function composition) 기반 프레임워크로 C#/.NET에 적합하다.
- AutoGen은 다중 에이전트 대화형 프로그래밍 프레임워크로 단순하나 디버깅이 어렵다.

---

## 5. Creating an agent to search the web

> LLM을 핵심으로 한 AI 검색은 매칭·랭킹을 넘어 맥락 이해, 개인화, 추론, 멀티모달을 통합한다.

- 전통 검색(PageRank)은 학습 없는 그래프 알고리즘으로 매칭과 랭킹 두 단계를 거친다.
- LLM은 키워드 의미·도메인을 구분하고 사용자 이력으로 개인화된 랭킹을 제공한다.
- 추출형 QA(extractive)와 추상형 QA(abstractive)로 링크 클릭 없이 답을 제공한다.
- 생성형 검색은 환각 위험이 있어 출처를 보존해 역추적(backtracking)이 가능해야 한다.
- ReAct(Reasoning and Acting) 프롬프팅은 추론 단계와 실행 단계를 결합한다.
- LangChain은 DuckDuckGo(무추적), Google Serper(저비용 API), Wikipedia 검색 도구를 제공한다.
- Tool에 이름·함수·설명을 주어 LLM이 어떤 도구를 쓸지 알게 하고 initialize_agent로 에이전트를 만든다.

### 웹 검색 에이전트 흐름

```
user query -> LLM analyzes & plans -> select tool (web search)
  -> retrieve documents -> LLM analyzes -> (more action? / done)
  -> generate answer -> user
```

---

## Summary (핵심 정리)

- LLM이 정교한 시스템의 두뇌가 되어 대화·추론 능력으로 작업을 해결함을 배웠다.
- 지각(senses)과 도구(hands)로 두뇌를 확장해 인터넷 검색과 멀티모달 입력을 처리하는 법을 익혔다.
- 다음 장에서는 모델이 메모리를 가지고 정보를 저장·검색하는 방법을 다룬다.
