# 07. Microsoft's GraphRAG Implementation

## 챕터 개요 (3줄 요약)
- MS GraphRAG는 LLM으로 entity·relationship을 추출·요약한 뒤, community를 detect·요약하는 2단계 indexing을 한다.
- 여러 chunk에 흩어진 정보를 entity/relationship/community 요약으로 통합해 일관된 지식 표현을 만든다.
- 검색은 global search(community 요약 map-reduce)와 local search(vector + graph traversal) 두 가지로 구현한다.

---

## 1. Dataset selection
> entity 정보가 풍부하고 여러 chunk에 걸쳐 분포된 데이터셋이 적합하다.

- entity type은 사전 정의(configurable). 보통 PERSON·ORGANIZATION·LOCATION + 도메인 특화(gene, legal clause 등).
- entity type 선택이 extraction·linking·summarization 품질 전체를 좌우.
- 예제: The Odyssey(Ulysses 등 핵심 entity가 여러 chunk에 등장).

---

## 2. Graph indexing (1단계: extraction & summarization)

### 2.1 Chunking
> 작은 chunk일수록 더 많은 entity를 추출하고, self-reflection 반복도 추출량을 늘린다.

- 자연 구분(24 books)으로 분할 후 token 수 확인 → 추가 chunking 필요.
- MS GraphRAG 연구: 600-token chunk가 2400-token보다 entity 추출 多. self-reflection 반복(추가 pass)도 누락 entity 추가 검출.
- 예: 1000-word, overlap 40으로 chunking.

### 2.2 Entity & relationship extraction
> LLM이 entity(name/type/description)와 related pair(source/target/description/strength score)를 delimited 포맷으로 추출한다.

- prompt(논문 appendix 차용): entity type 목록을 변수로 받아 entity 추출·분류·기술 → 관련 pair 식별·strength score 부여 → delimiter로 구조화 출력.
- The Odyssey entity types: PERSON, ORGANIZATION, LOCATION, GOD, EVENT, CREATURE, WEAPON_OR_TOOL.
  - PERSON/GOD는 명확, EVENT/LOCATION은 모호(유연성↑).
- 예: 66 entities, 182 relationships. 한 entity·pair가 여러 description 가짐(예: TELEMACHUS-MINERVA 14개 관계).

### 2.3 Entity & relationship summarization
> 같은 entity/relationship의 여러 description을 LLM으로 병합해 모순 해소·중복 제거한 단일 요약을 만든다.

- prompt: 여러 description을 3인칭·entity명 포함 단일 comprehensive 요약으로 통합, 모순은 해소.
- description >1인 entity/relationship만 요약 대상.
- **super node 주의**: 너무 많은 relationship을 가진 node(예: Athens)는 ranking/filtering으로 prompt token 관리 필요.

### 2.4 Community detection & summarization
> 밀집 연결된 entity 그룹(community)을 detect하고 community 단위 요약 report를 생성한다.

- community = 그래프 나머지보다 서로 더 densely 연결된 entity 그룹.
- **Louvain** 알고리즘 사용(논문은 Leiden, GDS 라이브러리). non-deterministic → 실행마다 약간 다름.
- 작은 그래프라 single-level만, hierarchical 구조는 생략(대형 그래프는 다층 granularity 가능).
- community summary prompt: TITLE, SUMMARY, IMPACT SEVERITY RATING(0-10), RATING EXPLANATION, DETAILED FINDINGS 구조.
- 대형 community는 ranking으로 핵심 entity·relationship만 선택.

---

## 3. Graph retrievers (2단계: retrieval)

### 3.1 Global search
> community 요약을 map-reduce로 집계해 데이터셋 전체를 아우르는 broad·thematic 질의에 답한다.

- **Map step**: 각 community report를 chunk화 → LLM이 key point list + importance score(0-100) 생성(JSON, report id 참조).
- **Reduce step**: 모든 intermediate response의 중요 point를 filter·aggregate → 최종 markdown 답변 합성.
- community hierarchy level 선택이 품질 좌우(저층=상세하나 LLM 호출↑, 고층=효율적이나 granularity↓).

```python
def global_retriever(query, rating_threshold=5):
    # rating >= threshold인 community 요약 조회
    # 각 community에 map prompt → intermediate_results
    # reduce prompt로 최종 답변 합성
```

### 3.2 Local search
> vector search로 관련 entity를 찾고, 연결된 entity·relationship·community report·text chunk를 graph traversal로 모아 entity-focused 질의에 답한다.

- entity-focused 질의(예: "chamomile 약효?")에 효과적.
- entity summary를 embed → vector index 생성 → query embedding으로 관련 entity 진입점 식별.
- local search Cypher: 진입 node에서 연결된 text chunk(frequency 순), community report(rank·weight 순), inside relationship, entity summary를 수집·ranking·limit.
- 수집 context를 string화해 local system prompt에 넣어 LLM이 최종 답변 생성(데이터 참조 표기, 없으면 모른다고 답).

```cypher
CALL db.index.vector.queryNodes('entities', $k, $embedding) YIELD node, score
WITH collect(node) as nodes
// text chunks / community reports / inside rels / entity summaries 수집·limit
```

---

## Summary (핵심 정리)
- MS GraphRAG는 entity·relationship 추출·요약 → community detection·요약의 2단계로 일관된 지식 표현을 만든다.
- 추출은 LLM이 사전 정의 type(PERSON, GOD 등)으로 entity를 분류하고 relationship strength score까지 부여한다.
- 여러 chunk의 description을 LLM 요약으로 통합해 중복 없는 단일 표현을 만든다.
- Louvain 등으로 densely 연결된 community를 detect하고 community 단위 요약으로 상위 테마를 포착한다.
- global search는 community 요약을 map-reduce로 broad 질의에 답한다.
- local search는 vector 유사도 + graph traversal로 entity-focused 질의에 답한다.
- chunk size·entity type·community 파라미터가 retrieval 효과를 좌우하며, 작은 chunk가 더 포괄적 추출로 이어진다.
- ranking 메커니즘으로 대규모 entity·relationship·community의 scaling 문제를 처리한다.
