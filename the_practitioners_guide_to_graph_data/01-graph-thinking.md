# 01. Graph Thinking

## 챕터 개요 (3줄 요약)
- 데이터베이스 기술의 역사(hierarchical → relational → NoSQL → graph)를 통해 graph thinking이 다시 부상한 배경을 설명한다.
- graph thinking은 데이터를 table이 아닌 relationship 중심으로 바라보며 complex system 안의 complex problem을 푸는 사고방식이다.
- 어떤 문제에 graph data가 필요한지, 그 relationship으로 무엇을 할지(analyze vs query, report/research/retrieval)를 결정하는 decision tree를 제시한다.

---

## 1. Why Now? 데이터베이스 기술의 4개 era

> 저장 효율 중심에서 데이터의 value 추출 중심으로 산업의 초점이 이동하면서 graph가 다시 relevant해졌다.

- **1960s–1980s Hierarchical/Navigational**: 데이터를 tree 구조로 저장, CODASYL이 세 가지 retrieval 목표(key, scan, link)를 표준화. link 탐색은 너무 느려 shelved, B-tree로 보완.
- **1980s–2000s Entity-Relationship (Relational)**: Codd의 relational algebra 기반. 데이터를 set/table로 조직, primary key로 접근, linking table로 entity 연결. "모든 데이터는 table에 매핑된다"는 사고를 정착시킴.
- **2000s–2020s NoSQL**: scale 가능한 비정형 데이터 처리. 동기는 serialization 표준(XML/JSON/YAML), 특화 tooling, horizontal scalability. key-value/wide-column/document/stream/graph 등장. scale-up → scale-out 전환.
- **2020s– Graph**: 저장 효율 → connected data에서 value 추출로 전환. CODASYL의 세 번째 목표(link traversal)로 산업이 회귀.

---

## 2. CODASYL의 세 가지 retrieval과 산업의 회귀

> key·scan은 이미 해결되었고, 마지막 남은 link traversal이 오늘날 graph technology의 핵심이다.

- key 접근: entity-relationship era에서 완성.
- scan 접근: NoSQL era에서 대규모 처리 가능.
- link traversal: 이제 value 추출 needs와 맞물려 graph로 full circle.

---

## 3. What Is Graph Thinking?

> graph thinking은 문제 도메인을 interconnected graph로 이해하고 graph 기법으로 도메인 동역학을 기술해 문제를 푸는 것.

- complex system: 다수의 component가 상호연결되어 단순 합 이상의 emergent behavior를 보이는 시스템.
- complex problem: complex system 안에서 관측·측정 가능한 개별 문제.
- 고가치 business problem 대부분이 complex problem이며 graph thinking을 요구.
- 4대 패턴: **neighborhoods, hierarchies(trees), paths, recommendations**.

---

## 4. Complex Problems in Business

> 데이터가 by-product가 아니라 ROI를 내는 strategic asset으로 바뀌면서, 도메인의 graph를 소유한 기업이 높은 가치를 가진다.

- Microsoft의 LinkedIn(26억×, $26B) / GitHub($7.8B) 인수 = professional graph / developer graph의 가치.
- Google(human knowledge), Amazon·FedEx(supply chain), Verizon(telecom), Facebook(social), Netflix(entertainment) 모두 도메인의 graph를 소유.

---

## 5. Decision Tree ①: graph data가 필요한가? (Figure 1-3)

> 데이터의 shape이 database/technology 선택을 top-down으로 이끌어야 한다.

```
DATA SHAPE        ->  DATABASE
Relational/table  ->  RDBMS        (retrieved by primary key)
Hierarchical/nested -> Document DB  (root by ID)
Graph/links       ->  Graph DB     (queried by pattern)
```

- **Q1 Does your problem need graph data?**: 원하는 정보의 shape(row vs nested vs relationships)을 판단.
- **Q2 Do relationships help understand the problem?**: yes/no면 명확, "maybe"면 문제가 너무 큰 것 → 분해 후 top으로 복귀.
- 흔한 misstep: graph가 보인다고 모든 component에 graph 적용할 필요 없음 → 일부는 table/document로 projection.

---

## 6. Decision Tree ②: relationship으로 무엇을 할 것인가? (Figure 1-4)

> graph data로 하는 일은 크게 analyze(분석) vs query(조회) 두 갈래로 나뉜다.

- **Q3 What will you do with relationships?**: analyze(중요 relationship 탐색) vs query(질문이 정해진 retrieval).
- query 경로 → 바로 graph database 사용.
- analyze 경로 → 결과 용도를 한 단계 더 정의(Q4).

---

## 7. Decision Tree ③: 결과를 어디에 쓸 것인가?

> graph 분석 결과의 end goal은 reports(BI), research(R&D), retrieval(데이터 제품) 세 가지이며, 이 책은 retrieval에 집중한다.

- **reports**: 전통적 BI/인텔리전스 (이 책 범위 아님).
- **research**: data science/ML R&D (이 책 범위 아님).
- **retrieval**: end user에게 서비스하는 data-driven product. latency·availability·personalization 요구. → **이 책의 초점**.
- 예: LinkedIn의 1st/2nd/3rd-degree connection 표시 = retrieval 경로의 graph metric.

---

## 8. 종합 decision process & 시작점

> 네 가지 질문(graph data 필요? / relationship이 도움? / 무엇을 할까? / 결과 용도?)을 합친 decision tree가 책 전체의 길잡이다.

- 모호하면 "분해하고 다시 시도"가 핵심 조언 — small하게 시작해 proven value 위에 쌓기.
- analysis paralysis 경계: R&D에서 production으로 넘어가지 못하는 실패를 피하라.

---

## Summary (핵심 정리)
- DB 역사 4 era(hierarchical → relational → NoSQL → graph)에서 value 추출 needs가 graph thinking을 다시 부상시켰다.
- graph thinking = 데이터를 table이 아닌 relationship 중심으로 보아 complex system 속 complex problem을 푸는 사고방식.
- 핵심 4 패턴: neighborhoods, hierarchies(trees), paths, recommendations.
- 두 단계 decision tree로 (1) graph data가 필요한가 (2) relationship으로 analyze/query 중 무엇을 하고 결과를 report/research/retrieval 중 어디에 쓸지 결정.
- 이 책은 graph database가 필요한 **retrieval(데이터 제품)** 경로에 집중한다.
