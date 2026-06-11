# 01. Improving LLM Accuracy

## 챕터 개요 (3줄 요약)
- LLM은 강력하지만 knowledge cutoff, hallucination, private data 부재 등 본질적 한계를 가진다.
- RAG는 외부 knowledge base에서 사실을 retrieval해 prompt에 주입함으로써 이 한계를 완화한다.
- knowledge graph는 structured + unstructured 데이터를 한 framework에 통합해 RAG의 이상적 저장소가 된다.

---

## 1. Introduction to LLMs
> LLM은 transformer 기반 next-token 예측 모델로, 사실을 저장하는 게 아니라 학습된 weight로 그럴듯한 응답을 생성한다.

- ChatGPT는 GPT-4 같은 LLM 위에 올린 conversational interface.
- 대량 텍스트로 pretraining → grammar·context·약한 reasoning 패턴 습득.
- 핵심: LLM은 "fact database"가 아니라 language의 수학적 표현(weights)을 학습한 것. 사실 회상이 아니라 next-word 예측.
- instruction-following 능력이 단순 autocomplete를 넘어 RAG pipeline 설계의 기반이 됨.

---

## 2. Limitations of LLMs
> LLM의 사실성 한계는 knowledge cutoff, outdated info, hallucination, private data 부재 네 가지로 정리된다.

- **Knowledge cutoff**: 학습 시점 이후 사건은 모름.
- **Outdated information**: cutoff 이전이라도 이후 변경된 사실(예: 소유권 변동)은 반영 못 함.
- **Pure hallucination**: 확신에 찬 어조로 fabricated 정보 생성. URL·citation·WikiData ID 같은 external reference에 특히 취약. 원인 = LLM은 reasoning engine이 아니라 probabilistic 패턴 매칭.
- **Lack of private information**: 사내/독점 데이터는 학습에 없어 답변 불가.
- 기타 한계(본서 범위 밖): bias, 실제 이해 부재, prompt injection 취약성, inconsistent 응답.

---

## 3. Overcoming the limitations of LLMs
> finetuning은 비용·신뢰성 문제로 한계가 있고, RAG가 더 단순하고 효율적인 해법이다.

- LLM 학습 4단계: pretraining → supervised finetuning → reward modeling → reinforcement learning.
- **Supervised finetuning**: 입력-정답 예시로 추가 학습. 하지만 pretraining은 비용 과다해 지속 업데이트 비현실적이고, finetuning으로 새 fact 학습은 연구상 효과가 엇갈림.
- **RAG (Retrieval-Augmented Generation)**: 외부 knowledge base와 LLM 결합. 두 단계 = retrieval(관련 정보 검색) + augmented generation(검색 정보를 prompt에 결합해 생성).
- 사용자는 질문만 제공; retrieval은 백그라운드에서 query 변환→검색→prompt template 결합 후 LLM 호출. ChatGPT의 Web Search도 RAG 사례.

---

## 4. Knowledge graphs as the data storage for RAG applications
> knowledge graph는 node(entity/concept)와 relationship으로 structured·unstructured 데이터를 한 곳에 담아 RAG 저장소로 이상적이다.

- knowledge graph = nodes(개념/entity) + relationships(연결).
- structured 데이터: 정확한 query(필터·count·aggregate) 가능 — 예: "누가 OpenAI CEO인가".
- unstructured 데이터(article text 등): 풍부한 context 제공하나 단독으로는 정밀 연산 불가.
- 둘을 같은 framework에 통합 → entity linking, structured 결과를 source passage로 맥락화 등 advanced retrieval 가능.

---

## Summary (핵심 정리)
- LLM은 transformer 기반 next-token 예측 모델로, 사실 저장이 아닌 학습된 weight로 응답을 생성한다.
- 핵심 한계: knowledge cutoff, outdated/hallucinated 정보, private·domain 지식 부재.
- 지속적 finetuning은 자원·복잡성 측면에서 비실용적.
- RAG는 외부 knowledge base의 사실을 prompt에 직접 주입해 정확성을 높이고 hallucination을 줄인다.
- 기존 RAG는 unstructured 데이터에 치중해 정밀·구조적 질의에 한계.
- knowledge graph는 node·relationship으로 structured + unstructured를 통합 표현.
- knowledge graph를 RAG에 결합하면 정확·신뢰·설명가능한 응답 생성이 가능해진다.
