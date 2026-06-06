# 13. Knowledge Graph–Powered Retrieval-Augmented Generation

## 챕터 개요 (3줄 요약)

- AI 에이전트는 LLM(두뇌)+프롬프트(안내)+도구(외부 세계 상호작용)의 조합으로, RAG(Retrieval-Augmented Generation)로 환각·정보 노후화·투명성·프라이버시 문제를 해결한다.
- 벡터 기반 RAG는 맥락 파편화·확장성·임베딩 한계·검색 노이즈·누락이라는 약점이 있다.
- Graph RAG는 KG를 LLM과 통합해 다중홉 관계 패턴으로 추론력과 검색 정밀도를 강화하고 투명성·통제력을 높인다.

---

## 1. AI Agents

> AI 에이전트는 환경과 상호작용하며 복잡한 작업을 수행하는 자율 엔티티로, 의사결정·학습·실시간 적응 능력을 보인다.

- "프랑스 수도?" 같은 단순 질문은 LLM이 답하나 ROI를 정당화하지 못하며, 실제 가치 있는 작업은 다단계 추론, 깊은 관계 패턴 이해, 최신·비공개 외부 데이터 접근을 요한다.
- 지식 컷오프(knowledge cutoff): LLM은 너무 커서 자주 재훈련 못 해 최신 정보·민감한 내부 데이터에 답하기 어렵다.
- 콘텐츠 작성 비서처럼 Researcher·Writer·Reviewer 같은 다중 에이전트가 역할극처럼 협력해 목표를 달성한다.

---

## 2. Chatting with the LLM

> 가장 단순한 챗봇 에이전트는 사전학습 LLM 접근과 대화 기억(memory)만 필요하다.

- Agent 클래스는 messages 인스턴스 변수를 기억으로 삼아 질문-답변 이력을 보존해 대화 경험을 제공한다.
- "cyclotron 자금 영향력자?" 첫 질문엔 일반적 답변(NIH·DOE 등), 맥락(1930년대 록펠러 재단)을 명시한 후속 질문엔 Ernest O. Lawrence·Vannevar Bush 같은 구체적 답변이 나온다.
- 즉, out-of-the-box 모델은 답변 근거 데이터를 보지 못하면 구체적 답을 못 준다.

---

## 3. Challenges in the Production Environment

> 실세계 에이전트 개발은 여러 LLM 관련 문제를 고려해야 한다.

- 환각(hallucination): 다음 토큰 예측 특성상 그럴듯하나 조작된 사실을 생성한다.
- 신선도(freshness/knowledge cutoff): 재훈련이 연 1~2회라 최신 발전을 반영 못 한다.
- 투명성(transparency): 답변 생성 과정·출처·신뢰도를 알 수 없다.
- 데이터 프라이버시, 비용(재훈련·배포의 재정·환경 비용), 윤리·편향(편향된 데이터 학습) 문제도 있다.

---

## 4. Chatting with the AI about Private Data

> LLM은 전문 도메인 지식이 제한적이라, 언어 이해력을 보존하면서 우리 비공개 데이터의 전문가로 만들어야 한다.

- 록펠러 아카이브 KG로 "cyclotron 연구 자금 영향력자?" 같은 질문에 비공개 데이터 기반으로 정확히 답할 수 있다.
- 목표: Cypher·차트·그래프 조작 없이 광범위한 사용자에게 가치를 제공하는 AI 인터페이스 — RAG를 사용한다.

### 13.4.1 Retrieval-Augmented Generation (RAG)

- RAG는 사전학습 LLM의 지식·언어 이해력에 질문 관련 외부 맥락(구조화 DB 또는 비정형 데이터)을 결합한다.
- LLM+프롬프트+도구(질문 관련 외부 정보 검색 함수)로 코딩되며, 모델 범위를 제공된 맥락으로 제한하는 그라운딩(grounding) 기법이다.
- 저자들은 인간을 대체가 아닌 증강하며 피드백 검증·감독으로 인간을 루프에 두는 것이 필수라 강조한다.
- 초기 RAG: 문서를 청크→임베딩(고정 길이 벡터)→벡터 DB 인덱싱, 질문도 임베딩해 가장 유사한 문서를 검색(LangChain·Neo4jVector 사용).

```
  Documents -> chunks -> embeddings -> vector DB
  Question -> embedding -> similarity search -> top-k docs -> LLM answer
```

### 13.4.2 Vector-based RAG Limitations

- 맥락 파편화로 인한 제한된 추론(문서를 독립 취급, 다중홉 관계 놓침, 청킹 전략 한계).
- 확장성(대규모 코퍼스에서 연산 비용 큼), 임베딩 한계(단일 벡터로 의미 과단순화, 희소성, 정적 벡터).
- 검색 노이즈(distraction, 무관 문서가 모델을 혼란), 검색 누락(가장 관련 있는 문서 누락).
- 예: "Lauritsen과 cyclotron 연구 관계?" 질문에 상위 3개 문서 중 하나만 "Lauritsen"을 언급(임베딩은 전체 의미를 인코딩해 특정 엔티티 언급을 보장 못 함).

### 13.4.3 Graph RAG

- KG를 LLM과 통합한 Graph RAG는 벡터 RAG 한계를 완화하며, KG가 원시 텍스트·메타데이터·구조화 소스를 통합하는 중심 지식 저장소가 된다.
- 록펠러 KG는 텍스트 속성 그래프(text-attributed, 노드·관계에 텍스트 속성)와 텍스트 짝 그래프(text-paired, 노드·관계가 원본 문서와 연결)의 결합이다.
- 활용: 메타데이터(날짜·타입·저자), KG 검색기(질문+스키마→Cypher 생성 또는 엔티티 연결 서브그래프), KG 강화 문서 검색기(모든 질문 엔티티를 언급하는 문서만), 결합 검색(다중 소스 질문 분할).

### 13.4.4 ~ 13.4.5 Reasoning Agents & Chatting with KG

- 도구 실행 순서가 불명확할 때 ReAct(Reason and Act) 에이전트가 추론→행동→관찰의 동적 피드백 루프로 문제를 푼다.
- LangChain의 create_structured_chat_agent로 KG 검색기·KG 강화 문서 검색기·벡터 검색(백업) 도구를 묶으며, 도구 이름·설명을 잘 작성하면 안정성·예측성이 크게 향상된다.
- 예: "동료들이 Dorothy M. Wrinch에 대해 뭐라 했나?"에 KG 검색기로 화자 목록을 얻고 KG 강화 문서 검색기로 관련 문서를 찾아 4단계 Thought/Action/Observation으로 정확히 답한다.
- "Harvard와 Johns Hopkins의 공유 연구 주제?" 같은 집계(aggregate) 질문은 여러 문서에 걸쳐 KG가 점들을 연결해 답하며, distraction·환각 위험을 줄이고 더 빠르고 저렴하다.

---

## Summary (핵심 정리)

- AI 에이전트의 핵심은 두뇌인 LLM, 안내하는 프롬프트, 외부 세계와 상호작용하는 도구의 조합이며, RAG는 생성 모델과 정보 검색을 결합해 환각·신선도·투명성·프라이버시 문제를 해결한다.
- 벡터 기반 RAG는 제한된 추론·확장성 문제·검색 부정확성(노이즈·누락)의 약점이 있다.
- Graph RAG는 KG의 구조적 다중홉 패턴으로 추론력·검색 정밀도를 강화하며, 텍스트 속성·텍스트 짝 그래프를 결합한 KG가 잘 큐레이션된 구조 지식과 문서·메타데이터를 함께 활용하게 한다.
