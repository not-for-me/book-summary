# 03. Adding Knowledge: Bass

## 챕터 개요 (3줄 요약)
- foundational model은 학습 데이터에 갇힌 closed system — knowledge cutoff·model capacity 한계·private data 접근 불가·hallucination·citation 불가 문제를 runtime에 RAG로 해결한다.
- RAG는 indexing(chunking+저장) → retrieval(관련 chunk 검색) → generation(context에 주입해 grounding)의 3단계 composable system이며, 이 chapter는 Pattern 6→7→8로 sophistication을 쌓아간다.
- 키워드 매칭(BM25)에서 → embedding 기반 Semantic Indexing → 대규모 운영(metadata·data freshness·contradiction 처리)으로 발전.

---

## 1. Pattern 6: Basic RAG
> indexing·retrieval·generation 3단계로, 관련 chunk를 prompt에 주입해 LLM 응답을 trusted source에 grounding한다.

- **해결 문제**: static knowledge cutoff, model capacity limit(lossy compression), private data 접근 불가, hallucination, citation 불가.
- **grounding 원리**: LLM은 context의 정보를 우선 사용 → priming effect로 context 문장이 학습 지식을 override(예: Geno Smith가 Raiders 소속). 관련 chunk는 query를 알아야 식별 가능 → runtime compute.
- **2 pipeline**: indexing pipeline(batch — Document를 SentenceSplitter로 chunk=node화, overlap 추가, metadata 부착, docstore 저장) + question-answering pipeline(retrieval→generation).
- **retrieval**: TF-IDF → 흔한 stop word 제거, rare term 가중. **BM25**는 term saturation(count/(count+k))과 확률적 relevance로 TF-IDF 개선.
- **generation**: system(instruction) + system(context nodes) + user(query) 메시지를 LLM에 전송, 응답에 citation용 node 동봉.
- **한계**: exact keyword match 의존("ruptured" vs "broken" 다른 결과), chunk size 제약(긴 chunk = 고비용·느림). RAG vs large context window(작은 문서는 전체 주입 + Prompt Caching이 나을 수 있음).

```
[Indexing]  sources ─▶ chunk(node) + metadata ─▶ docstore
[QA]  query ─▶ retriever(BM25) ─▶ top-k nodes ─▶ prompt context ─▶ LLM ─▶ grounded answer + citations
```

## 2. Pattern 7: Semantic Indexing
> chunk의 의미를 embedding vector로 encode해 vector store에 색인 — 키워드가 아닌 meaning으로 query와 chunk를 매칭한다.

- **키워드 한계**: synonym/pronoun("AI"≠"Artificial Intelligence"), 전체 의미 누락, cross-language, multimodal, layout context, false positive(CHF 중의성).
- **embedding**: 의미적으로 유사한 콘텐츠를 vector space에서 가깝게 배치. dimensionality는 표현력 vs 계산효율(curse of dimensionality, N²) 균형 — 최소 차원·차원축소·ANN 근사.
- **semantic chunking**: length+overlap, sentence-based, paragraph-based, document-structure(Markdown heading), semantic shift 기반(topic modeling).
- **multimodal**: 이미지는 OCR/LLM 캡션→텍스트화 또는 multimodal embedding; 비디오는 transcript + keyframe 샘플링.
- **table**: table-based / sliding window(header 부착) / row-based / column-based chunking, table metadata 보존.
- **industry jargon**: synonym expansion(query 또는 문서 확장), glossary·cooccurrence·LLM(directionality 주의: ETF⊂index fund).
- **contextual retrieval**: 작은 chunk에 문서 요약 context를 prepend (Anthropic prompt로 오검색 67% ↓, document 부분은 Prompt Caching).
- **hierarchical chunking(RAPTOR)**: chunk→cluster→summary를 재귀적으로 트리화, inference 시 트리 순회로 다양한 granularity 제공 (GraphRAG 단순화 버전).
- **한계**: fixed-dim 정보 병목, chunking이 cross-reference 단절, vector DB 확장성(ANN trade-off), temporal 이해 부족, reasoning 불가, text/image vector space 정렬 문제, tabular data가 text embedding에 압도됨.

## 3. Pattern 8: Indexing at Scale
> 대규모 production RAG에서 disambiguation·data freshness·contradiction·model lifecycle을 metadata로 다루는 전략 집합.

- **문제**: disambiguation(fluid의 물리/일상 의미), data freshness(CDC 격리 지침 변화), contradictory information(고혈압 기준 140/90↔130/80 변천), model lifecycle(embedding 모델 deprecation 시 전체 reindex).
- **metadata 종류**: document-level(source·timestamp·author·topic·복잡도), chunk-level(위치·entity·semantic role·language), domain-specific(API version·SKU·jurisdiction), auth/confidentiality(접근 권한·암호화·redaction).
- **contradiction 탐지**: temporal tagging(timestamp 차이), source 신뢰도, subject 분류, version tracking으로 최신·권위 있는 chunk 선택.
- **outdated content 처리**: retrieval filtering(날짜 제한), document store pruning(오래된 chunk 삭제 — index 축소로 빠름), result reranking(최신·신뢰 source boost).
- **model lifecycle**: embedding API deprecation(보통 12개월+)에 대비 — 긴 지원 lifecycle API 또는 open-weight 모델(local/hyperscaler 호스팅), MTEB 랭킹 참고. 효율 향상·최신 cutoff 필요 시에만 교체.
- **한계/대안**: metadata 품질 의존, binary filter 제약, temporal relevance 단순 날짜의 함정 → domain-specific index 분리·incremental indexing·semantic 관계 유지.

---

## Summary (핵심 정리)
- foundational model의 knowledge cutoff·capacity·private data·hallucination·citation 문제를 runtime에 RAG로 해결 (CPT/retraining은 비용·빈도 비현실적).
- **Basic RAG**: indexing→retrieval(BM25)→generation, context grounding(priming effect). 한계: exact match·chunk size.
- **Semantic Indexing**: embedding으로 meaning 기반 매칭, semantic/hierarchical(RAPTOR) chunking, multimodal·table·jargon·contextual retrieval 처리.
- **Indexing at Scale**: metadata로 disambiguation·data freshness·contradiction·model lifecycle 관리.
- RAG vs large context window: 작은 corpus(~200K token)는 전체 주입 + Prompt Caching이 RAG 복잡성보다 나을 수 있음.
- 다음 chapter(Syncopation)에서 retrieval 품질을 더 끌어올리는 패턴 지속.