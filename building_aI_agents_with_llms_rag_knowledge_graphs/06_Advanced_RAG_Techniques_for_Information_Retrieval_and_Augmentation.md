# 06. Advanced RAG Techniques for Information Retrieval and Augmentation

## 챕터 개요 (3줄 요약)

- naïve RAG의 한계를 해결하는 고급 RAG(검색 전/후 최적화) 기법들과 모듈형 RAG를 다룬다.
- 계층적 인덱싱, HyDE, 하이브리드 검색, 쿼리 라우팅, 리랭킹 등 구성 요소와 실습 파이프라인을 설명한다.
- 빅데이터 확장성, 보안·프라이버시, 장문맥 LLM·멀티모달 RAG 등 미래 과제까지 살펴본다.

---

## 1. Discussing naïve RAG issues

> naïve RAG는 검색·증강·생성 각 단계에서 정밀도, 누락, 형식, 확장성 등 여러 문제를 가진다.

- 검색 도전(retrieval challenge): 정밀도·재현율 부족과 노후화된 지식 베이스 문제가 있다.
- 상위 랭크 문서 누락: 관련 청크가 top-k에 들지 못하거나 임베딩 모델이 약할 수 있다.
- 컨텍스트 부족: 답이 담긴 문서가 너무 많아 컨텍스트 길이에 안 들어간다.
- 추출 실패: 올바른 컨텍스트가 있어도 노이즈·충돌 정보로 맥락 환각(contextual hallucination)이 발생한다.
- 잘못된 형식·특이성: LLM이 불릿·표 요청을 무시하거나 답이 너무 모호/기술적일 수 있다.
- 정보 중복(redundancy)과 불완전한 답변, 유연성 부족 문제가 있다.
- 확장성·성능: 임베딩·생성이 느리거나 보안이 취약할 수 있다.

---

## 2. Exploring the advanced RAG pipeline

> 고급 RAG는 검색 전(pre-retrieval)과 검색 후(post-retrieval) 단계를 최적화해 naïve RAG 문제를 해결한다.

- 계층적 인덱싱(hierarchical indexing)은 각 계층 요약을 임베딩해 정확도를 높이며 map-reduce 변형이 있다.
- 가설 질문(hypothetical questions)은 청크별 가상 질문을 임베딩하고, HyDE는 가상 답변을 임베딩해 검색한다.
- 컨텍스트 강화: 문장 윈도우 검색(sentence window), 부모 문서 검색기(parent document retriever).
- 쿼리 변환(query transformation): 분해, 스텝백 프롬프팅, 재작성, 확장(query expansion).
- 하이브리드 검색은 BM25 키워드 검색과 벡터 검색을 alpha 가중치(보통 0.3~0.5)로 결합한다.
- 쿼리 라우팅(query routing): 논리·키워드·제로샷·함수 호출·시맨틱 라우터로 흐름을 제어한다.
- 리랭킹(reranking)은 크로스 인코더, 멀티 벡터(ColBERT), LLM(pointwise/pairwise/listwise)로 청크를 재정렬한다.
- 컨텍스트 압축(LongLLMLingua, autocompressor)과 반복/재귀/적응형(Flare, Self-RAG) 검색이 있다.

### 가설 질문 vs HyDE

```
Hypothetical Qs: chunk -> LLM generates question -> embed question -> match query
HyDE          : query -> LLM generates answer  -> embed answer   -> match chunks
```

---

## 3. Modular RAG and its integration with other systems

> 모듈형 RAG는 검색·메모리·라우팅·생성·검증 등 특화 모듈을 순차/병렬로 재구성하는 유연한 패러다임이다.

- 모듈형 RAG는 "retrieve and read"를 넘어 "retrieve, read, and rewrite"를 가능케 한다.
- DSP(Demonstrate-Search-Predict)와 Self-RAG는 LLM과 RAG의 복잡한 상호작용·자기비판을 보여준다.
- 학습 없는(training-free) 방식은 임베더와 LLM을 동결한 채 사용한다.
- 독립 학습(independent training)은 검색기와 LLM을 따로 파인튜닝한다.
- 순차 학습(sequential): retriever-first(검색기 먼저)와 LLM-first(LLM 감독으로 검색기 학습).
- LLM-first는 큰 모델의 지식을 작은 검색기로 전이하는 지식 증류와 유사하다.
- 결합 학습(joint training)은 검색기와 생성기를 동시에 정렬하는 end-to-end 방식이다.

---

## 4. Implementing an advanced RAG pipeline

> 리랭커, 쿼리 변환·라우팅, 하이브리드 검색, 요약 같은 add-on을 naïve RAG에 추가해 고급 파이프라인을 만든다.

- 쿼리 변환은 동의어·관련어를 추가해 검색 범위를 넓힌다.
- 쿼리 라우팅은 특정 키워드 유무에 따라 텍스트 검색과 벡터 검색을 선택한다.
- 하이브리드 검색(fusion_retrieval)은 벡터 결과와 Elasticsearch 키워드 결과를 결합한다.
- 리랭커는 BERT로 청크 관련도를 재정렬해 가장 관련 있는 청크를 컨텍스트에 넣는다.
- 컨텍스트 압축은 LLM 요약기로 중복 정보를 제거해 노이즈를 줄인다.
- 이들 컴포넌트를 하나의 파이프라인으로 조립해 사용한다.

---

## 5. Understanding the scalability and performance of RAG

> 프로덕션 환경에서 RAG는 빅데이터, 병렬 처리, 보안·프라이버시 측면의 도전에 직면한다.

- 데이터 유형: 비정형(텍스트·이미지), 반정형(PDF·JSON·HTML), 정형(SQL·Excel·KG)을 다뤄야 한다.
- chain-of-table은 CoT와 테이블 변환을 결합해 복잡한 표를 처리한다.
- 데이터 레이크(data lake), 파티셔닝, 캐싱, 중복 제거(deduplication)로 빅데이터를 관리한다.
- 병렬 처리(Apache Spark, Dask)로 저장·검색·생성 단계의 지연을 줄인다.
- 보안: AES-256, TLS/SSL 암호화, MFA(다단계 인증), GDPR·CCPA 준수가 필요하다.
- RAG 특화 공격: 임베딩 역전(embedding inversion), 프롬프트 주입, RAG 포이즈닝, 멤버십 추론(MIA).
- NeMo Guardrails(Colang)와 Llama Guard로 입력·출력·검색·대화 레일을 적용한다.

### 빅데이터 RAG 확장 전략

```
storage  -> distributed / data lake / partitioning / caching
retrieval-> shard across nodes (parallel)
generation-> tensor/model parallelism
monitoring-> accuracy + memory + cost + latency
```

---

## 6. Open questions and future perspectives

> 장문맥 LLM, 멀티모달 RAG, 맥락 환각 등이 RAG의 미해결 과제이자 미래 전망이다.

- 장문맥 LLM(LC-LLM)은 100K~100만 토큰을 처리하나 중간 정보 활용이 비효율적이고 비용·환각이 크다.
- LC-LLM은 RAG를 대체하지 못하며, 임베더는 최대 32K 토큰만 처리 가능하다.
- 멀티모달 RAG 전략: 동일 공간 임베딩(CLIP), 단일 기준 모달리티, 모달리티별 분리 검색.
- 맥락 환각(contextual hallucination)은 올바른 컨텍스트가 있어도 LLM이 자체 지식을 쓸 때 발생한다.
- 모델 확신도(confidence)가 높을수록 컨텍스트보다 자체 답을 고수하며, 프롬프트 강도로 조절 가능하다.
- 환각 감소 요인: 데이터 품질, 맥락 인식, 부정 거부(negative rejection), 추론 능력, 도메인 정합성.
- 강화학습 적용, 그래프 검색(GraphRAG) 통합, 인터넷 통합 등이 활발히 연구된다.

---

## Summary (핵심 정리)

- naïve RAG의 문제점과 이를 해결하는 고급 RAG add-on, 그리고 모듈형 RAG 패러다임을 배웠다.
- 빅데이터 확장, 병렬 처리, 보안·프라이버시 등 프로덕션 도입 시의 도전 과제를 익혔다.
- 장문맥 LLM·멀티모달 같은 미해결 과제를 살펴봤으며, 다음 장에서 지식 그래프(KG)와 GraphRAG를 다룬다.
