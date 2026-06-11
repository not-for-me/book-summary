# 13. Epilogue

## 챕터 개요 (3줄 요약)
- graph thinking 여정을 정리하고, relational과 graph 관점을 모두 익혀 문제를 분해·결합하라고 권한다.
- "development first, production second" 접근과 5가지 핵심 패턴(neighborhood/tree/path/collaborative filtering/entity resolution)을 Lego 블록처럼 조합하라고 강조한다.
- 추가 학습 방향 4가지(graph algorithms, distributed graphs, graph theory, network theory)와 추천 자료를 제시한다.

---

## 1. 마무리 — graph thinking의 위치

> graph는 relational보다 낫지 않고 "다르다" — 특정 문제 클래스에 더 쉽고 효과적.

- 숙련은 지속적 연습으로 — notebook을 자신의 문제에 맞춰 활용.
- 초반엔 relational(tabular) 관점이 편하지만, relationship·연결 구조가 중요할 때 graph thinking을 시도.
- 복잡한 문제는 보통 두 관점의 조합으로 분해 — 양쪽 모두 숙달이 중요.

---

## 2. Development First, Production Second

> 데이터를 graph로 탐색·반복하며 기법을 정제한 뒤 production 튜닝으로.

- Ch4~12의 5대 연결 데이터 패턴: exploring neighborhoods, branching in trees, finding paths, collaborative filtering, entity resolution.
- 이 기법들을 Lego 블록처럼 조합해 애플리케이션 솔루션 구축.

---

## 3. 추가 학습 방향 4가지

> graph thinking은 시작의 끝일 뿐.

- **Graph algorithms**: 전체 graph 구조 분석(PageRank, connected components, betweenness centrality 등). 일부는 localized 분해 가능, 대부분(PageRank 등)은 distributed batch 필요.
- **Distributed graphs**: 너무 크거나 워크로드·geo 요구로 분산. consistency(DataStax는 eventual consistency, 가용성 우선) 이해 중요.
- **Graph theory**: graph 구조의 수학(planar graph, graph coloring 등). 용어·개념 이해.
- **Network theory**: graph theory의 실세계 적용(소셜/생물 네트워크). 자연 네트워크 다수가 **scale-free**(power law degree 분포) → supernode 존재 이유. preferential attachment("rich get richer") 이론.

---

## 4. 추천 자료 & 연결 유지

> 각 방향별 심화 도서.

- Graph Algorithms (Needham & Hodler, O'Reilly) / Distributed Graph Algorithms (Erciyes).
- Cassandra: The Definitive Guide (Carpenter & Hewitt) / Principles of Distributed Database Systems (Özsu & Valduriez).
- Introduction to Graph Theory (Trudeau) / Linked (Barabási), "Structure and Function of Complex Networks" (Newman).
- Twitter @Graph_Thinking, GitHub datastax/graph-book.

---

## Summary (핵심 정리)
- graph는 relational보다 낫지 않고 다름 — relationship이 중요할 때 더 효과적, 양쪽 관점 숙달이 복잡한 문제 해결의 핵심.
- "development first, production second"로 탐색·반복 후 튜닝. 5대 패턴(neighborhood/tree/path/collaborative filtering/entity resolution)을 Lego처럼 조합.
- 추가 학습 4방향: graph algorithms(전역 계산), distributed graphs(분산·consistency), graph theory(수학), network theory(scale-free·preferential attachment).
- 자연 네트워크는 대부분 scale-free → supernode가 존재하는 근본 이유.
