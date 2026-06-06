# 01. Introducing LLMs, RAGs, and Neo4j Knowledge Graphs

## 챕터 개요 (3줄 요약)

- GenAI(Generative AI)와 LLM(Large Language Model)의 발전 흐름, 영향력, 그리고 환각·윤리적 한계를 개관한다.
- LLM의 한계를 보완하는 핵심 기법으로 RAG(Retrieval-Augmented Generation)와 knowledge graph를 소개한다.
- Neo4j knowledge graph가 LLM 응답을 사실 기반으로 grounding하여 multi-hop reasoning을 가능케 하는 방식을 제시한다.

---

## 1. Outlining the evolution of GenAI through the lens of LLMs

> 2022년 말 ChatGPT 등장으로 GenAI가 대중화되었으며, 규칙 기반 → 머신러닝 → 딥러닝/LLM으로 진화해 왔다.

- AI는 1990년대 rule-based system에서 machine learning 알고리즘으로, 다시 deep learning과 LLM으로 진화했다.
- OpenAI의 GPT-3가 대중의 관심을 처음으로 끈 general-purpose LLM이었다.
- GenAI는 텍스트·이미지·비디오 생성 등 다양한 modality를 다루지만, 본 책은 비즈니스/산업 use case의 LLM에 집중한다.
- LLM은 NLP(Natural Language Processing)를 위한 머신러닝 모델로, 학습 기반으로 언어 구조를 이해하고 콘텐츠를 생성한다.
- BERT(Bidirectional Encoder Representations from Transformers), GPT(Generative Pre-trained Transformer)가 LLM을 선도한 대표 연구다.
- LLM은 본질적으로 RNN(Recurrent Neural Network) 계열이며, 장기 의존성 문제를 LSTM(Long Short-Term Memory)이나 transformer로 해결한다.

### 기본 LLM 아키텍처 (Figure 1.1)

```
[Input Layer] -> [Embedding Layer] -> [Encoder (LSTM/Transformer)] -> [Decoder] -> [Output]
  prompt           word vectors          contextual encoding           word-by-word generation
```

- Input layer는 초기 텍스트 prompt/sequence를 받고, Embedding layer는 단어를 의미를 담은 numerical vector로 변환한다.
- Encoder는 multi-layered RNN(LSTM) 또는 transformer로 문맥 정보를 포착하고, Decoder는 출력 sequence를 한 단어씩 생성한다.
- GPT 파라미터 규모: GPT-1(117M) → GPT-2(1.5B) → GPT-3(175B) → GPT-4 series(170T)로 버전마다 수 차수씩 증가한다.

### Understanding GenAI's pitfalls and ethical concerns

> LLM은 언어를 진짜로 이해하지 못하고 다음 token을 예측할 뿐이라, 환각·윤리적 문제가 발생한다.

- LLM은 사실·감정·윤리를 이해하지 못하고 학습 텍스트의 패턴만 인식하여 텍스트를 생성한다.
- 존재하지 않는 판례를 만들어낸 legal brief 사례처럼, 그럴듯하지만 사실이 아닌 콘텐츠를 생성할 수 있다.
- 공격적 이미지·비디오 생성 등 사회적·법적·윤리적으로 부적절한 결과물이 생길 수 있다.
- 해로운/부정확한 콘텐츠를 식별하고 재학습 또는 별도 검증 장치로 대응해야 한다.

---

## 2. Understanding the importance of RAGs and knowledge graphs in LLMs

> GenAI의 한계는 fine-tuning 또는 외부 소스로 응답을 grounding하여 보완할 수 있다.

- Fine-tuning은 기존 모델에 추가 정보를 학습시켜 고품질 응답을 얻지만, 복잡하고 시간이 많이 든다.
- RAG는 LLM에게 질문할 때 외부 지식 저장소에서 검색한 추가 정보를 제공하여 응답을 grounding한다.
- 활용 소스: 공개 구조화 데이터셋(PubMed, Wikipedia), 기업 knowledge base(내부 문서, 제품 카탈로그), 도메인 특화 소스(법률 판례, 의료 가이드라인).
- RAG는 정적 학습 데이터와 달리 real-time 검색으로 데이터 최신성·정확성·특수성 문제를 해결한다.
- knowledge graph는 RAG를 위한 또 다른 핵심 정보원으로, 구조화·상호연결된 기반을 제공한다.

### The role of knowledge graphs in LLMs

> knowledge graph는 데이터를 다차원·동적으로 표현하여 AI 결과를 풍부한 맥락으로 grounding한다.

- knowledge graph는 고정된 단일 차원이 아니라 temporal·spatial·contextual 정보를 live data feed로 실시간 포착한다.
- Enhanced contextual understanding: 고립된 사실이 아닌 관계 기반 검색(예: 의료에서 증상-질병-치료 연결).
- Efficient data retrieval: multi-hop reasoning으로 여러 단계 떨어진 통찰 도출(예: 금융의 숨은 관계).
- Integration of vector embeddings: vector embedding의 의미 유사성과 결합해 정확성·관련성을 높인다.
- Real-world impact: e-commerce는 리뷰·구매이력·제품특성 등 다양한 소스로 맥락 풍부한 추천을 제공한다.

---

## 3. Introducing Neo4j knowledge graphs

> Neo4j는 데이터를 graph로 저장하는 데이터베이스로, 데이터와 관계의 진화에 따라 knowledge graph가 동적으로 발전한다.

- Neo4j는 노드에 multiple label과 optional schema 접근을 지원하여 데이터 의미 변화를 유연하게 반영한다.
- 데이터 의미가 진화하면 추가 label이나 특정 relationship으로 노드 간 맥락을 persist(유지)할 수 있다.
- 매장에서 프로모션 상품을 전면에 배치하듯, knowledge graph도 유연하게 변화를 포착해야 한다.
- Neo4j knowledge graph는 LLM 응답을 사실 기반으로 grounding하는 데 활용된다.

### Using Neo4j knowledge graphs with LLMs (Figure 1.2 — Healthcare 예시)

> 의료 챗봇이 환자 증상 기록(구조화)과 연구 논문·임상시험(비구조화)을 Neo4j로 연결해 medical reasoning을 강화한다.

- 비구조화 텍스트는 Ollama, OpenAI, Hugging Face 등의 embedding 모델로 처리 후 NER(Named Entity Recognition)로 증상·치료 등 핵심 entity를 추출한다.
- Neo4j knowledge graph에서 문서는 증상·치료를 mention하고, 환자는 증상을 show하며, 증상은 잠재적 치료와 link된다.
- 이를 통해 "독감과 유사 증상을 보이면서 과거 COVID-19 증상도 보인 환자는?" 같은 복합 질의를 multi-hop으로 답할 수 있다.

```
[Research Documents] --mention--> [Symptoms] --linked_to--> [Treatments]
        |                              ^
        v                              |
   [embedding + NER]              [Patients] --show--> [Symptoms]

Query path:
1. flu와 연결된 symptoms 검색 (research documents)
2. 해당 symptoms를 보이는 patients 식별
3. 과거 기록에서 COVID-19 symptoms 교차 참조
4. 두 조건 모두 충족하는 patients 반환 (출처 문서 포함)
```

- multi-hop knowledge graph query path로 grounding하여 사실적으로 정확하고 최신인 의료 의사결정 지원 결과를 생성한다.

---

## Summary (핵심 정리)

- GenAI는 LLM의 발전과 함께 대중화되었으나, 사실·윤리 측면의 환각 한계를 지닌다.
- RAG와 knowledge graph는 외부 검증 정보로 응답을 grounding하여 LLM의 정확성과 reasoning을 향상시키는 핵심 enabler다.
- Neo4j knowledge graph는 구조화·비구조화 데이터를 연결하고 multi-hop reasoning을 가능케 하여, 다음 장의 RAG 심화로 이어진다.
