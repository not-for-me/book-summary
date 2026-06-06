# 01. How to Get Value from Graphs in Just Five Days

## 챕터 개요 (3줄 요약)

- 연결된 데이터(connected data)의 가치는 관계(relationship)에 있으며, 그래프 데이터베이스는 사일로(silo)된 데이터를 조직의 디지털 트윈(digital twin)으로 통합해 데이터를 민주화한다.
- 관계형 데이터베이스(RDB)의 임피던스 불일치(impedance mismatch)와 조인(JOIN) 성능 한계를 그래프가 어떻게 해결하는지, 그리고 UBO·실시간 추천·법 집행·사이버 범죄 등 대표 활용 사례를 소개한다.
- 가상의 음악 스트리밍 회사 ElectricHarmony를 사례로, Neo4j와 Cypher를 사용해 단 5일 만에 추천 시스템 개념증명(PoC, Proof of Concept)을 점진적으로 구축하는 실습 여정을 보여준다.

---

## 1. Why Graph Databases?

> 관계형 데이터베이스는 실제 비즈니스 도메인을 테이블로 평탄화(flatten)했다가 조회 시점에 조인으로 재조립하기 때문에 성능과 인지적 부담이 모두 발생하는데, 그래프는 이 불일치를 자연스럽게 해소한다.

- 애플리케이션은 데이터를 객체 그래프(object graph)로 다루지만 RDB는 테이블로 저장하므로 임피던스 불일치(impedance mismatch)가 발생한다.
- 관계형 DB는 관계 표현을 조인 테이블(join table)에 의존해, 상호 연결성이 높은 데이터에서 다중 조인으로 인한 성능 병목이 생긴다.
- 그래프 모델은 화이트보드에 그린 비즈니스 도메인과 거의 동일해 비기술 이해관계자도 직관적으로 이해한다.
- 그래프는 데이터 사용 방식을 명확히 하고, 데이터의 공백이나 의미 오해를 드러낸다.
- 데이터와 복잡도가 커질수록 그래프의 이점은 커진다 — 정규화 규칙·다수 테이블·인덱스 관리 같은 부가 복잡도를 더하지 않기 때문이다.
- GenAI(Generative AI) 시대에 지식 그래프(knowledge graph)는 명시적 관계를 포착해 LLM(Large Language Model)을 검증된 사실에 근거하게 한다(GraphRAG).

---

## 2. Graph Use Cases

> 그래프는 거의 모든 도메인에 맞지만, 관계로부터 가치를 끌어내는 활용 사례에서 진가를 발휘한다.

- 최종수익소유자(UBO, Ultimate Beneficial Ownership): KYC(Know Your Customer) 절차의 핵심으로, 수십 계층의 깊고 순환적인 소유 구조를 효율적으로 추적해 자금세탁방지(AML)를 지원한다.
- 실시간 추천(Real-Time Recommendations): 고객·상품·서비스·소셜 관계를 그래프로 순회하여 콘텐츠/협업 필터링을 빠르게 수행한다.
- 법 집행(Law Enforcement): 사람·사건·장소 간 숨은 연결을 드러낸다.
- 사이버 범죄 네트워크(Cybercrime Networks): 공격 패턴과 연관 엔티티를 추적한다.
- 추천에 쓰이는 그래프 데이터 사이언스 알고리즘에는 커뮤니티 탐지(community detection, 고객 세분화), 유사도(similarity) 알고리즘, 링크 예측(link prediction)이 포함된다.

---

## 3. Neo4j and Cypher

> Neo4j는 네이티브 그래프 데이터베이스(native graph database)이며, Cypher는 ASCII 아트 기반의 선언형(declarative) 그래프 질의 언어다.

- Cypher는 2011년 Neo4j가 만들었고, 2017년 openCypher 오픈소스 구현으로 이어졌다.
- Cypher에서 파생된 GQL(Graph Query Language)은 SQL에 이어 ISO/IEC가 표준화한 두 번째 데이터베이스 질의 언어다.
- Cypher는 "어떻게(how)"가 아니라 "무엇을(what)" 찾거나 생성할지를 기술하는 선언형 언어다.
- 노드는 괄호 `()`, 관계는 선 `--`과 방향 화살표 `<>`로 표현하는 시각적 언어다.

```
( :Artist ) -[ :CREATED ]-> ( :Album ) -[ :CONTAINS ]-> ( :Track )
   node          relationship   node                       node
```

---

## 4. The Song Recommendation System: A Five-Day Proof of Concept

> ElectricHarmony 팀에 합류해 5일 동안 점진적으로 음악 추천 PoC를 구축하며, 빅뱅(big bang)이 아닌 반복적(incremental) 가치 전달 방식을 체험한다.

### Day 1 — Install, Ingest, Model
- Neo4j(LTS 5.26 기준) 로컬 설치 또는 관리형 클라우드 Neo4j Aura(Free/Professional/Enterprise) 사용을 선택한다.
- 첫 데이터셋을 적재하고 미리보기한 뒤 그래프 모델(Track·Album·Artist 등)을 설계하고 그래프를 순회(walk)한다.

### Day 2 — Nodes, Relationships, and Safe MERGE
- `LIMIT 1`로 한 행만 적재하는 드라이런(dry run)으로 모델 타당성을 먼저 검증한다.
- `MERGE`를 사용해 중복(duplicate)을 안전하게 방지하며 데이터셋 간 질의를 수행한다.

### Day 3 — Indexes, Data Quality, Similarities
- 적재 속도 향상을 위해 인덱스(index)를 추가한다 — 인덱스가 없으면 약 1만 행 적재도 매우 느려질 수 있다.
- 최소 데이터 품질(minimum data quality)을 확보하고 플레이리스트 간 유사도를 계산한다.

### Day 4 — Materializing Similarities
- 매 요청마다 계산하지 않고, 유사한 플레이리스트 사이에 `SIMILAR` 관계와 유사도 점수를 명시적으로 생성(materialize)한다.
- 암묵적(implicit) 관계를 명시화해 조회 성능을 높인다.

### Day 5 — Recommendations and Explainability
- 제한된 데이터셋에서 추천 결과를 생성하고, 그래프 기반의 설명 가능성(explainability)으로 왜 특정 곡이 추천되었는지 추적한다.
- 폐쇄형 블랙박스가 아니라 추적 가능한 추천이라는 점이 그래프의 강점이다.

---

## Summary (핵심 정리)

- 그래프 데이터베이스는 관계형 DB의 조인/임피던스 불일치 문제를 해결하고, 연결된 데이터를 통해 조직 전체가 동일한 언어로 데이터를 다루도록 민주화한다.
- Neo4j와 선언형 질의 언어 Cypher를 이용하면 설치→적재→모델링→유사도→추천의 5일 PoC를 점진적으로 구축할 수 있다.
- 이후 챕터들은 이 PoC를 넘어 대규모 적재(2장), 쿼리 튜닝(5장), 트랜잭션 쓰기 경로(9장) 등 프로덕션 모범 사례로 확장된다.
