# 04. Adding Knowledge: Syncopation

## 챕터 개요 (3줄 요약)
- Ch3의 Basic RAG/Semantic Indexing 위에 retrieval 품질·신뢰성·복잡 질의 처리를 끌어올리는 4개 고급 패턴(9~12)을 쌓는다.
- "chicken-and-egg" 문제(답을 모르는데 어떻게 매칭하나)를 hypothetical answer·query expansion·hybrid search·GraphRAG로 해결.
- retrieval 후 reranking·compression·disambiguation(Node Postprocessing), 신뢰 구축(Trustworthy Generation), 반복적 search-think-reason(Deep Search)으로 발전.

---

## 1. Pattern 9: Index-Aware Retrieval
> chunk가 무엇을 담고 어떻게 색인됐는지를 활용해 Semantic Indexing의 매칭 한계를 보완한다 (hypothetical answer·query expansion·hybrid search·GraphRAG).

- **문제**: 질문이 KB에 없음, KB가 query와 다른 기술 용어 사용("Muslim palace" vs "Nasrid fortress"), 답이 chunk 속 미세 detail, 답이 여러 chunk의 holistic 해석 필요.
- **Hypothetical answers (HyDE)**: query 대신 LLM이 만든 가상 답변으로 chunk 매칭 (답이 틀려도 OK). 관점 다양화로 perspective 문제도 처리.
- **Query expansion**: query에 context·용어 번역 추가(비기술 query → 기술 KB 매칭). HyDE와 결합 가능.
- **Hybrid search**: BM25 + embedding 점수의 weighted average(alpha; 0=BM25, 1=vector). 큰 chunk의 세부 손실 보완.
- **GraphRAG**: graph DB에 node 트리로 색인, 관계로 related chunk 검색. LLMGraphTransformer로 entity 추출, Neo4j 저장.
- **고려사항**: hypothetical answer/expansion은 foundational model 지식 기반 → 미지 domain에선 hallucination·obsolete·irrelevant("Alexander의 패턴" → 건축가 Christopher Alexander) 위험, intent 왜곡 가능.

## 2. Pattern 10: Node Postprocessing
> retrieval과 generation 사이에 단계를 삽입해 relevance↑·ambiguity↓ 및 업데이트·개인화를 처리 — 핵심은 reranking.

- **문제**: similarity≠relevance(목차가 retrieve됨), chunk 내 무관 정보, ambiguous entity(Colorado vs Yellowstone Grand Canyon), 충돌·obsolete, generic 답변.
- **reranking**: LLM/BGE로 (query, chunk) relevance score 산출 — embedding보다 정확(단일 vector 압축 안 함). 단, LLM call·latency·비용↑ (= LLM-as-Judge).
- **hybrid search**: 여러 retriever 결과를 합쳐 reranker에 통과 (점수 비교 불가 문제 해소).
- **query expansion/decomposition**: retriever별 다른 query, subpart 분해, 2차 검색.
- **filtering**: metadata로 최신 연도만 유지(obsolete 제거). 충돌 탐지는 N*(N-1) call로 고비용.
- **contextual compression**: relevance score 계산 LLM call에 무관 텍스트 제거를 folding (call 절약).
- **disambiguation**: 첫 chunk와 나머지 비교(N-1 call)로 동명이의 entity 탐지 → follow-up 질문.
- **personalization/대화 이력**: user context·과거 거래·conversation summary를 context에 추가.
- **고려사항**: reranking은 runtime 비용·latency↑(BGE 같은 소형 모델로 완화). 작업 많으면 단일 LLM call에 folding + structured output.

## 3. Pattern 11: Trustworthy Generation
> retrieval failure·context 신뢰성·reasoning error·hallucination을 완전히 없앨 수 없으므로, 신뢰도를 평가·전달하는 기법 집합.

- **out-of-domain detection**: query-chunk embedding distance 임계값, zero/few-shot 분류, domain 키워드 요구 → 하이브리드(가중합). 감지 시 short-circuit 또는 라우팅(Google Maps).
- **citations**: ① source-level tracking(metadata lineage, over-cite 경향), ② classification-based(common knowledge vs 인용 필요 구분, 정밀하나 복잡), ③ token-level attribution(attention으로 토큰별 출처, paraphrase 대응, 연구 단계).
- **guardrails**: 검색 전(harmful query 필터·injection sanitize), 검색 후(metadata·신뢰 source 우선·privacy), 생성 전(freshness·source 다양성), 생성 후(citation 강제·fact-check·harmful 점검). 도구: Guardrails AI, DeepEval, Ragas.
- **observability**: context/response relevance·faithfulness·recall/precision 추적 (Arize Phoenix, Langfuse 등).
- **human feedback**: chunk up/down vote, 응답 review queue, explicit/implicit feedback로 embedding·LLM fine-tune.
- **CRAG**: 생성 전 evaluator가 chunk 품질 평가 → 무관 시 web/enterprise 검색 augment 또는 decompose-then-recompose.
- **self-RAG (reflection)**: self-evaluation·adaptive retrieval·controlled generation으로 생성 결과를 자기 비판 (prompt로 구현, LLM-as-Judge).
- **UI design**: citation link·source preview·confidence meter·progressive disclosure·thumbs feedback로 black box→검증가능 assistant.
- **한계**: 임계값이 domain-specific·튜닝 필요, safeguard 과하면 정보 손실, human 검증 확장 한계.

## 4. Pattern 12: Deep Search
> search-read-reason의 반복 루프로 복잡한 query에 포괄적 답변을 생성 — RAG에 thinking 단계·iteration·외부 tool을 추가한다.

- **문제**: context window 제약, query 중의성, 정보 staleness·검증, 얕은 reasoning, multihop query(한 검색 결과로 다음 query 형성).
- **3대 확장**: ① retrieval과 generation 사이 thinking 단계(추가 정보 결정), ② 단일이 아닌 다단계 retrieval-generation iteration(state 유지), ③ KB뿐 아니라 search engine·enterprise API 활용.
- **gap 식별**: LLM으로 답변의 logical/information gap 탐지 → 최대 3개 follow-up query 생성, 없으면 빈 리스트.
- **iterative refinement**: 단순 질문부터 답하고 iteration마다 gap 채움.
- **evaluation metrics**: relevance·comprehensiveness·accuracy·coherence·citation·efficiency의 가중평균 (Ragas 권장, reference dataset로 검증).
- **information integration**: cross-document entity resolution, 모순 탐지·perspective clustering, source 신뢰도 ranking, temporal reasoning.
- **Deep Search vs Deep Research**: 전자는 간결한 답, 후자는 long-form report (출력 누적 방식만 차이).
- **예시**: Wikipedia 기반 — search→rank→extract→synthesize→thinking(gap·follow-up)→재귀적 section 트리(`deep_search(query, depth)`).
- **고려사항**: 다중 iteration으로 매우 느림·고비용 → subquery 병렬처리·조기 종료로 완화.

---

## Summary (핵심 정리)
- **Index-Aware Retrieval**: HyDE·query expansion·hybrid search·GraphRAG로 question-chunk 매칭의 chicken-and-egg 문제 해결.
- **Node Postprocessing**: reranking 중심으로 hybrid search·compression·disambiguation·filtering·personalization (similarity≠relevance 교정).
- **Trustworthy Generation**: out-of-domain 감지·citation·guardrail·human feedback·CRAG·self-RAG·UI로 신뢰 구축.
- **Deep Search**: thinking·iteration·외부 tool로 복잡·multihop query에 포괄적 답변 (Deep Research는 long-form 변형).
- 공통: 대부분 단계가 LLM prompt로 구현 가능하나 latency·비용 trade-off를 신중히 평가해야.
- Ch3-4의 패턴은 Basic RAG(6)부터 sophistication이 단계적으로 증가하는 add-on 구조.