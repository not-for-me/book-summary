# 12. Semantic Search and Similarity

## 챕터 개요 (3줄 요약)
- 자연어 문서는 구조가 없어 단순 키워드 검색은 한계가 크다 — NLP(Natural Language Processing)로 엔터티를 추출해 그래프로 만들면 "문자열이 아닌 사물(things, not strings)" 검색이 가능해진다.
- NER(Named-Entity Recognition)로 문서를 주석(annotation)하고 salience(중요도) 가중치로 연결하면, 엔터티 기반 검색·트렌드 분석·문서 유사도 추천이 가능하다.
- 엔터티를 조직 원리(온톨로지/개념 스킴)에 연결(엔터티 링킹/disambiguation)하면 "NoSQL DB" 검색에 직접 언급 없는 Neo4j 기사도 매칭하는 진정한 시맨틱 검색이 완성된다.

## Search over Unstructured Data
> 초기 검색은 인덱스 키워드 매칭에 불과해 어형 변화·약어·오타에 취약하고 랭킹도 단순 빈도뿐이었다.

- Google이 두 축으로 혁신: PageRank(하이퍼링크로 랭킹) + Knowledge Graph(의미 보강).
- 지식그래프는 동의어·인접 키워드·도메인 개념을 해소해 검색을 "질의응답" 영역까지 확장.

## From Strings to Things: Entity Annotation (NER)
> NER은 텍스트 속 숨은 엔터티를 추출 — "New York Times"(ORG)와 "New York"(LOC)을 같은 단어라도 구별한다.

- 도구: Hugging Face(bert-base-NER: LOC/ORG/PER/MISC), 결과는 entity·type·salience(중요도) 반환.
- 그래프 저장: `(Article)-[:references {salience}]->(Entity)` — "이 문서는 무엇에 관한가?" / "이 엔터티를 언급한 문서는?" 양방향 질의.
- 단어 빈도(word count)가 아닌 "엔터티 빈도"로 의미있는 인기도·트렌드 시계열 분석.

```
// 엔터티 추출 결과 저장
MERGE (a:Article {url:$url}) SET a.title=$title
UNWIND $entityList AS entity
MERGE (e:Entity {name:entity.word, type:entity.type})
MERGE (a)-[:references {salience:entity.score}]->(e)
```

## Document Similarity for Recommendations
> 두 문서가 공유하는 엔터티의 salience 가중 합이 유사도 — 콘텐츠 기반 추천("You may also like X")의 토대.

- `(a1)-[r1:references]->(e)<-[r2:references]-(a2)` 패턴으로 공유 엔터티 탐색, `sum(r1.salience*r2.salience)`로 유사도 계산.
- 설명 가능(explainable) 추천: 공유 엔터티를 함께 반환. 대규모는 :similar 관계로 사전 물질화(materialize)해 조회 가속.
- Cold start(앵커 없음): 최근 트렌딩 토픽 표시 또는 유사도 알고리즘 사전 계산으로 완화.

## Making Annotation Semantic with an Organizing Principle
> NER만으로는 엔터티 중의성 해소·관계 부재의 한계 — 조직 원리(온톨로지)에 링킹하면 의미가 명시화된다.

- 엔터티 링킹(disambiguation): GCP Natural Language API는 Wikipedia URL 메타데이터 제공 → Wikidata/DBPedia(상호참조됨)의 항목과 매칭.
- neosemantics(n10s)로 OWL/SKOS/RDFS 조직 원리를 그래프에 임포트, APOC NLP로 추출+링킹.
- 결과: "NoSQL database management system" 검색이 `(c:Concept)<-[:broader*0..]-(sc)<-[:refers_to]-(article)`로 Neo4j(graph DB ⊂ NoSQL) 기사까지 반환 = "things, not strings".
- 사례 NASA LLIS: 2천만 문서 silo를 엔터티로 연결, 검색 결과 1,100건→소수 정밀 결과, "화성 미션 연구개발 1년·$2M 절감".

> [모델링 관점 - 주식시장 도메인 적용]
> 이 장은 주식시장의 "비정형 데이터(뉴스·공시·리포트·실적발표 전사)"를 그래프로 끌어들이는 핵심 기법이다. 적용: (1) 금융 뉴스/공시에 NER+엔터티 링킹을 적용해 기업·인물·이벤트(M&A, 소송, 규제)를 추출하고 기존 비즈니스 그래프(기업-지분-섹터)의 노드에 연결 → "이 기업에 관한 모든 뉴스/공시"를 엔터티 기반으로 정밀 조회. 단, 금융은 동음이의(애플=Apple Inc. vs 과일, 삼성전자 vs 삼성생명)가 많으므로 ticker/LEI 기반 disambiguation이 필수. (2) salience 가중 유사도로 "유사 테마/연관 기업 뉴스" 추천 및 뉴스 클러스터링. (3) 조직 원리로 FIBO 같은 금융 온톨로지를 얹으면 "통화정책" 검색에 금리·환율·중앙은행 관련 문서가 직접 언급 없이도 매칭됨 — 온톨로지 기반 의미 검색이 바로 사용자가 목표한 "의미있는 인사이트 도출"의 직접 구현. (4) 뉴스 엔터티 빈도 시계열로 "테마 부상/소멸" 트렌드를 정량 추적. 이 장의 NER→링킹→온톨로지 파이프라인이 주식시장 지식그래프에 비정형 정보를 통합하는 표준 절차다.

## Summary (핵심 정리)
- NER로 문서에서 엔터티를 추출해 salience 가중 그래프로 만들면 엔터티 기반 검색·트렌드·설명가능 유사도 추천이 가능하다.
- 조직 원리(온톨로지)에 엔터티를 링킹하면 중의성 해소와 도메인 관계가 명시화되어 "직접 언급 없는" 시맨틱 검색이 완성된다.
- 주식시장에서는 뉴스·공시를 NER+ticker/LEI 링킹으로 비즈니스 그래프에 통합하고 FIBO 온톨로지로 의미 검색을 구현하는 것이 비정형 정보 활용의 정석이다.
