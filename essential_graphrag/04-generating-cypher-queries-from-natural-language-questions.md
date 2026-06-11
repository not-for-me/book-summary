# 04. Generating Cypher Queries from Natural Language Questions (text2cypher)

## 챕터 개요 (3줄 요약)
- text2cypher는 자연어 질문을 Cypher query로 변환해 graph DB에서 실행하는 retrieval 방식이다.
- vector search가 못 하는 aggregation·정밀 조건 질의를 처리하는 보완·"catchall" retriever 역할을 한다.
- prompt에 schema + terminology mapping + format instructions + few-shot examples를 넣어 정확도를 높인다.

---

## 1. The basics of query language generation
> 자연어 질문을 graph DB에서 실행 가능한 Cypher query로 변환하며, 핵심 난제는 정확하고 적합한 query 생성이다.

- 대부분 LLM은 Cypher 기본 문법을 안다. 핵심 = 질문 의미 + graph schema 이해.
- schema 미제공 시 LLM은 label·relationship·property 이름을 추측할 수밖에 없음.
- schema 제공 = 질문 의미와 graph model(어떤 label/relationship type/property/연결)의 mapping 역할.
- workflow: 질문 수신 → schema 조회 → terminology mapping/format/few-shot 정의 → prompt 생성 → LLM이 Cypher 생성.

---

## 2. Where it fits in the RAG pipeline
> vector search가 못 하는 aggregation·특정 조건 질의에 text2cypher를 사용한다.

- 예: "Spielberg 감독 영화 top3 평점과 평균" → vector similarity로는 불가, aggregation Cypher 필요.

```cypher
MATCH (:Reviewer)-[r:REVIEWED]->(m:Movie)<-[:DIRECTED]-(:Director {name:'Steven Spielberg'})
RETURN m.title, AVG(r.score) AS avg_rating ORDER BY avg_rating DESC LIMIT 3
```

- 가장 유사한 node 검색이 아니라 데이터 집계·특정 추출이 필요할 때 사용.
- agentic 시스템에서 다른 retriever와 매칭 안 될 때 "catchall" 역할 가능.

---

## 3. Useful practices for query language generation
> few-shot examples, schema, terminology mapping, format instructions 네 가지가 정확도의 핵심이다.

- **Few-shot examples (in-context learning)**: 질문-Cypher 쌍 예시 제공 → LLM이 패턴 학습. graph별 수작업 작성. 반복되는 schema 오해(property로 읽을지 traversal할지 등)를 교정.
- **Database schema in prompt**: 사용 가능한 label·relationship·property 명시. Neo4j는 APOC `apoc.meta.data()`로 schema 추론(큰 DB는 sampling). 포맷 자체는 성능에 큰 영향 없음(Neo4j 내부 연구).
- **Terminology mapping**: 질문 용어 ↔ schema 용어 연결(예: "actor/director" → `Person` label). graph별 특화, 시간이 지나며 진화.
- **Format instructions**: 출력 형식 통일(설명·사과 금지, Cypher만 출력, code block 금지).

---

## 4. Implementing a text2cypher generator (base model)
> schema + terminology + format + few-shot을 하나의 prompt template로 조립해 LLM에 전달한다.

```text
Instructions: Generate Cypher statement ...
Graph database schema: {schema}
Terminology mapping: {terminology}
Examples: {examples}
Format instructions: ONLY RESPOND WITH CYPHER—NO CODE BLOCKS.
User question: {question}
```

- 예: "Who directed the most movies?" → 생성 결과:

```cypher
MATCH (p:Person)-[:DIRECTED]->(m:Movie)
RETURN p.name, COUNT(m) AS movieCount ORDER BY movieCount DESC LIMIT 1
```

---

## 5. Specialized (finetuned) LLMs for text2cypher
> Neo4j는 text2cypher 전용 finetuned 모델·데이터셋을 공개하며, 큰 모델보다 효율적이다.

- Neo4j가 Hugging Face에 text2cypher 학습 데이터 + Gemma2/Llama 3.1 기반 finetuned 모델 공개.
- 최신 GPT/Gemini보다 성능은 뒤처지나 훨씬 효율적 → 큰 모델이 느린 production에 적합.
- few-shot·schema·terminology·format으로 추가 개선 가능.

---

## Summary (핵심 정리)
- query language generation은 aggregation·특정 데이터 추출 시 다른 retrieval을 보완하는 RAG 구성요소다.
- 유용한 실천: few-shot examples, schema, terminology mapping, format instructions.
- base model로 prompt를 구조화해 text2cypher retriever를 구현할 수 있다.
- text2cypher 전용 finetuned LLM을 쓰고 위 기법으로 성능을 개선할 수 있다.
