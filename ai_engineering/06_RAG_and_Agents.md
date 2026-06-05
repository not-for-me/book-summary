# 06. RAG and Agents

## 챕터 개요 (3줄 요약)
- 쿼리별 관련 context를 구성하는 두 패턴인 RAG(외부 메모리에서 정보 검색)와 agent(도구 사용)를 다룬다.
- RAG의 retriever 구조(term-based vs embedding-based), 최적화 기법, 텍스트 너머(멀티모달·tabular) 확장을 설명한다.
- agent의 환경·도구·planning·reflection, 실패 모드와 평가, 그리고 memory 시스템을 정리한다.

---

## 1. RAG (Retrieval-Augmented Generation)
> 외부 메모리에서 관련 정보를 검색해 모델 생성을 보강하는 기법으로, context 한계를 넘고 hallucination을 줄인다.

긴 context가 RAG를 대체하진 못한다(데이터는 계속 증가, 긴 context를 잘 활용 못함, 비용·latency). 쿼리별 맞춤 context 구성으로 classical ML의 feature engineering에 해당한다.

### RAG Architecture
> retriever(외부 메모리 검색)와 generator(검색 정보 기반 생성)로 구성된다.

retriever는 indexing(나중 검색용 데이터 처리)과 querying(관련 데이터 검색) 기능을 한다. 문서를 chunk로 나눠 인덱싱하고, 쿼리마다 관련 chunk를 검색해 프롬프트에 결합한다.

### Retrieval Algorithms
> 문서를 쿼리 관련성으로 랭킹하며, term-based와 embedding-based로 나뉜다.

**Term-based**(sparse, lexical): TF(Term Frequency)와 IDF(Inverse Document Frequency)를 결합한 TF-IDF, BM25(문서 길이 정규화)가 대표. Elasticsearch는 inverted index 사용. 빠르고 out-of-box 강하나 term ambiguity 문제.

**Embedding-based**(dense, semantic): 데이터를 embedding으로 변환해 vector database에 저장, 쿼리 embedding과 가까운 chunk를 검색. vector search는 ANN(Approximate Nearest Neighbor) 알고리즘 사용: LSH, HNSW, Product Quantization, IVF, Annoy 등. finetuning으로 개선 가능하나 비용이 크다.

평가 지표: context precision(검색된 것 중 관련 비율), context recall(관련 것 중 검색된 비율), 랭킹은 NDCG/MAP/MRR. **Hybrid search**: term-based + embedding-based를 순차(reranking)나 병렬(RRF, Reciprocal Rank Fusion)로 결합.

```
Query -> Embedding model -> [vector search in vector DB] -> top-k chunks -> Generator
```

### Retrieval Optimization
> chunking, reranking, query rewriting, contextual retrieval 기법.

**Chunking**: 고정 길이(문자/단어/문장/단락), 재귀적 분할, overlap으로 경계 정보 보존. chunk 크기는 trade-off(작으면 다양하나 정보 손실·오버헤드 증가). **Reranking**: 검색 후 재정렬(시간 기반 등). **Query rewriting**: 모호한 후속 질문을 독립적으로 재작성. **Contextual retrieval**: chunk에 metadata·요약·원문 맥락을 덧붙여 검색성 향상.

### RAG Beyond Texts
> 멀티모달(이미지 등)과 tabular 데이터로 확장된다.

멀티모달은 CLIP 같은 multimodal embedding으로 텍스트·이미지를 함께 검색한다. tabular 데이터는 text-to-SQL → SQL 실행 → 응답 생성 워크플로를 쓴다.

---

## 2. Agents
> 환경을 인지하고 행동하는 존재로, AI가 task를 분석·계획·실행하는 두뇌 역할을 한다.

agent는 환경과 사용 가능한 action(도구로 확장)으로 정의된다. 멀티스텝으로 compound mistakes가 누적되고 stakes가 높아 더 강한 모델이 필요하다.

### Tools
> 도구는 agent가 환경을 인지(read)하고 행동(write)하게 한다.

세 범주: **knowledge augmentation**(retriever, web browsing 등 context 구성), **capability extension**(계산기, code interpreter, 번역기 등 한계 보완·멀티모달화), **write actions**(이메일 발송, DB 변경, 송금 등 환경 변경). write action은 강력하나 보안 위험이 크다. 모델 제공자들은 function calling으로 도구 사용을 지원한다.

### Planning
> 복잡한 task는 목표와 제약을 이해해 단계 roadmap(plan)을 세워야 한다.

planning과 execution을 분리해 검증된 plan만 실행한다(heuristic이나 AI judge로 검증). 과정: plan generation(task decomposition) → reflection/error correction → execution(function calling) → reflection. intent classifier가 도구 선택을 돕는다. 사람이 어느 단계든 개입 가능하다. LLM의 planning 능력은 논쟁적이나, world model로 action 결과를 예측하면 가능하다는 견해도 있다.

```
Task -> Plan generation -> [validate?] --no--> regenerate
                              |yes
                           Execute (function calling) -> Reflect -> done? --no--> replan
```

### Plan Generation, Function Calling, Granularity, Complex Plans
> prompt engineering으로 plan generator를 만들고 control flow를 다룬다.

function calling: tool inventory 선언 → 쿼리별 사용 도구 지정(required/none/auto). planning granularity는 hierarchical(고수준→저수준)로 trade-off 회피하며, 함수명 대신 자연어 plan이 도구 변경에 robust하다(단 translator 필요). control flow: sequential, parallel, if statement, for loop. **Reflection**(ReAct: Thought-Act-Observation, Reflexion)은 필수는 아니나 성능을 크게 높인다.

### Tool Selection & Failure Modes
> 도구가 많을수록 능력은 커지나 사용이 어려워진다.

ablation study, 도구 호출 분포 분석으로 도구를 선택한다. 실패 모드: **planning failure**(invalid tool, 잘못된 파라미터, goal/제약 위반, reflection 오류), **tool failure**(올바른 도구가 잘못된 출력, 도구 부재), **efficiency**(단계 수·비용·시간). 각 실패 모드의 빈도를 측정해 평가한다.

---

## 3. Memory
> 모델이 정보를 보유·활용하는 메커니즘으로, RAG·agent에 특히 유용하다.

세 가지: **internal knowledge**(학습된 지식, 모든 쿼리 접근), **short-term memory**(context, 빠르나 용량 제한, task 간 비유지), **long-term memory**(외부 데이터, retrieval로 접근, task 간 유지·삭제 가능). 사용 빈도에 따라 배치한다.

이점: 세션 내 정보 overflow 관리, 세션 간 정보 유지(개인화), 일관성 향상, 구조적 무결성 유지. 두 기능: memory management(add/delete)와 memory retrieval(RAG retrieval과 유사). 관리 전략: FIFO(단순하나 중요 초기 정보 손실 위험), 요약·named entity 추적으로 redundancy 제거, reflection 기반(삽입/병합/교체). 모순 처리(최신 우선 or AI 판단)는 use case에 따른다.

---

## Summary (핵심 정리)
- RAG는 외부 메모리에서 검색 후 생성하는 2단계 패턴으로, retriever 품질이 핵심이며 term-based(가벼운 baseline)와 embedding-based(vector search, 잠재력 큼)를 hybrid로 결합한다.
- agent는 환경과 도구로 정의되며 AI가 planner로 task를 분석·계획·실행하고, reflection과 memory로 능력을 보강하나 자동화될수록 실패·보안 위험이 커진다.
- RAG·agent 모두 대량 정보를 다뤄 context를 초과하므로, internal/short-term/long-term memory 시스템이 필요하다.
