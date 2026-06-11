# 03. Advanced Vector Retrieval Strategies

## 챕터 개요 (3줄 요약)
- 기본 vector search는 query-document 간 용어·맥락 차이로 recall이 부족할 수 있다.
- query rewriting(step-back prompting)으로 질문을 일반화하고, parent document retriever로 정밀 매칭 + 풍부한 맥락을 동시에 확보한다.
- Neo4j graph에 PDF→Parent→Child 계층 구조를 저장해 child embedding으로 매칭하되 parent 전체를 반환한다.

---

## 1. Step-back prompting (query rewriting)
> 구체적 질문을 더 일반적인 step-back 질문으로 재작성해 vector retrieval 정확도·recall을 높인다.

- 문제: user query embedding이 정작 정보가 담긴 document embedding과 용어·맥락 차이로 어긋날 수 있음.
- **query rewriting 전략**: hypothetical document retriever, step-back prompting.
- **step-back**: "2007~2008년 Thierry Audel 소속팀?" → "Thierry Audel 커리어 전체?"로 broaden → 더 넓은 검색.
- LLM이 rewriting에 적합(학습 불필요, prompt 지시만으로 가능).
- **few-shot prompting**: system prompt에 2~5개 예시 포함 → zero-shot보다 일관·신뢰성↑.

```python
def generate_stepback(question):
    return chat(messages=[
        {"role":"system","content":stepback_system_message},
        {"role":"user","content":question}])
```

---

## 2. Advanced embedding strategies
> 검색할 원문을 그대로 embed하지 않고, 문서 의미를 더 잘 대표하는 형태를 embed할 수 있다.

- **Hypothetical question**: 문서가 답할 수 있는 질문들을 embed → user query와 질문 embedding을 매칭 → 해당 문서 반환.
- **Parent document retriever**: parent를 작은 child chunk로 분할해 child만 embed → 매칭은 child로, 반환은 parent 전체(full context).
  - 장점: 긴 문서 전체 embed 시 averaging으로 개념이 흐려지는 문제 해소, 정밀 매칭 + 풍부 맥락 양립.
- **기타 정확도 개선**: embedding model finetuning(도메인 데이터, 단 전체 재임베딩 필요), reranking(2차 정렬), metadata 기반 filtering(저자·날짜·태그), hybrid retrieval(keyword+dense).

---

## 3. Parent document retriever 구현
> PDF를 section→parent(≤2000자)→child(500자) 계층으로 나눠 Neo4j graph에 저장하고, child embedding에 vector index를 건다.

- 텍스트를 구조 요소(paragraph/section) 기준으로 우선 분할 → 일관성 유지.
- regex로 numbered section 분리(예: 9개 section), tiktoken으로 token 수 측정(긴 section은 추가 분할).
- 데이터 모델: `(:PDF)-[:HAS_PARENT]->(:Parent)-[:HAS_CHILD]->(:Child {text, embedding})`.
- `:Child.embedding`에 vector index 생성.

```cypher
MERGE (pdf:PDF {id:$pdf_id})
MERGE (p:Parent {id:$pdf_id+'-'+$id}) SET p.text=$parent
MERGE (pdf)-[:HAS_PARENT]->(p)
UNWIND range(0,size($children)-1) AS i
MERGE (c:Child {id:...}) SET c.text=$children[i], c.embedding=$embeddings[i]
MERGE (p)-[:HAS_CHILD]->(c)
```

- **retrieval**: child를 `k*4`개 검색(safety buffer) → `HAS_CHILD`로 parent traverse → parent 중복 제거 → max score로 정렬 → 최종 k개 parent 반환.

```cypher
CALL db.index.vector.queryNodes($index_name, $k*4, $question_embedding) YIELD node, score
MATCH (node)<-[:HAS_CHILD]-(parent)
WITH parent, max(score) AS score
RETURN parent.text AS text, score ORDER BY score DESC LIMIT toInteger($k)
```

---

## 4. Complete RAG pipeline
> step-back prompting + parent document retrieval을 하나의 파이프라인으로 결합한다.

```python
def rag_pipeline(question):
    stepback = generate_stepback(question)
    docs = parent_retrieval(stepback)
    return generate_answer(question, docs)
```

- generation 함수는 ch02와 동일(system + user message에 문서·질문 전달).
- step-back으로 검색하되 원래 질문으로 답변 생성.

---

## Summary (핵심 정리)
- query rewriting은 user query를 target 문서의 언어·맥락에 맞춰 retrieval 정확도를 높인다.
- hypothetical document retriever·step-back prompting은 의도와 내용 간 격차를 메워 누락을 줄인다.
- 원문 그대로가 아닌 요약·paraphrase·hypothetical question을 embed해 문서 본질을 더 잘 포착할 수 있다.
- parent document retrieval은 작은 child로 정밀 매칭하고 parent로 풍부한 맥락을 제공해 답변 정확도를 높인다.
- 문서를 작은 chunk로 분할하면 granular한 retrieval이 가능해 특정 query가 가장 관련된 부분을 찾는다.
