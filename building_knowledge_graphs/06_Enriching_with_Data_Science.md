# 6. Enriching Knowledge Graphs with Data Science

## 챕터 개요 (3줄 요약)
- 그래프 데이터 과학(GDS, Graph Data Science)은 그래프 알고리즘으로 위상(topology)에 숨은 인사이트(영향력자·커뮤니티·경로·유사성)를 추출한다.
- Neo4j GDS는 "프로젝션(projection) → 메모리 적재 → 알고리즘 실행 → 결과 저장" 4단계로 동작하며, Cypher/Python으로 비전문가도 65+ 알고리즘을 쉽게 실행한다.
- 핵심 교훈: 알고리즘 결과는 종종 직관과 반대(counterintuitive)이므로, 가설을 빠르게 반복 검증하고 더 풍부한 데이터/프로젝션으로 정제해야 한다.

## Why Graph Algorithms? / Classes
> 그래프 알고리즘은 구조에서 정량적 인사이트를 끌어내며, 설계는 전문 영역이지만 "사용"은 목적과 문법만 알면 된다.

- 3대 범주: Statistical(노드·관계 수, 차수 분포 등 맥락 제공), Analytical(잠재 패턴 발굴), Machine Learning(7장).
- 분석 알고리즘 5종: Network propagation(전파 경로 — 질병·공급망 약점), Influence/Centrality(다리·병목 노드), Community detection(약한 링크 제거로 군집), Similarity(유사 패턴), Link prediction(누락 관계 예측).
- 같은 입력도 알고리즘마다 결과 상이(예: WCC vs Louvain) → 문서를 읽고 맥락에 맞게 선택·실험.

## Graph Data Science Operations
> GDS는 그래프 DB와 긴밀히 통합되어, 프로젝션을 만들고 멀티 CPU로 알고리즘을 실행한 뒤 결과를 그래프에 되써넣을 수 있다.

- 4단계: Read projected graph(관심 부분 선택) → Load(압축 인메모리) → Execute(알고리즘) → Store(write/stream/mutate).
- 결과 처리 모드: `.stream`(호출자에 반환), `.mutate`(프로젝션만 갱신), `.write`(기반 그래프 보강).
- 스케일링: scale-out(분산)은 locality 이점 상실로 비효율 → GDS는 단일 대용량 메모리 scale-up(CPU+RAM)이 더 빠르고 저렴. GPU는 선형대수로 표현 가능한 알고리즘에만 유리(보편 아님).

```
CALL gds.graph.project.cypher('g',
  'MATCH (p:Person) RETURN id(p) AS id',
  'MATCH (a:Person)-[:FRIEND]->(b:Person)
   RETURN id(a) AS source, id(b) AS target, "FRIEND" AS type');
CALL gds.betweenness.write('g', {writeProperty:'betweennessCentrality'});
```

## Experimenting with Graph Data Science
> Python API(graphdatascience) + Jupyter로 드라이버·세션 같은 DB 세부사항을 추상화해 데이터 과학자가 객체 수준에서 빠르게 실험한다.

- 영국 철도망 사례: Station 노드 + TRACK 관계(distance 속성) 적재 → trains 프로젝션 생성.
- Dijkstra 최단경로(Birmingham↔Edinburgh = 298), betweenness 중심성 계산.
- 반직관적 결과: 최고 중심성 역은 승객 많은 Birmingham New Street가 아니라 인근의 Tamworth(중심성 약 8배). 알고리즘이 승객·노선이 아닌 "연결된 선로"만 가중했기 때문 — 실제 네트워크 관점에선 Tamworth가 중요하며 장애 시 파급(network effect) 발생.

## Production Considerations / Enriching
> 운영에서는 워크로드를 역할별로 분리(primary=트랜잭션, secondary=GDS 연산)하고 결과를 그래프에 되써 시스템을 개선한다.

- Primary(트랜잭션, RAM·I/O 중심) / Secondary(비동기 갱신, 다중 CPU·RAM — 데이터 과학용)로 분리해 경합 방지. Neo4j는 단일 클러스터 HTAP(ETL/OLAP 큐브 불필요).
- causal barrier(bookmark)로 secondary에서도 "최소 자기 쓰기" 신선도 보장.
- 운영 전환은 `stream`을 `mutate`/`write`로 바꾸면 끝 — 결과를 그래프에 적재해 ML 피처 엔지니어링 등 후속 활용.

> [모델링 관점 - 주식시장 도메인 적용]
> 5대 알고리즘 범주는 주식시장 지식그래프에서 직접적 가치를 가진다: (1) Network propagation → 한 기업/섹터의 악재가 공급망·지분 체인을 따라 어디까지 전파되는지(시스템 리스크). (2) Centrality → 시장에서 "구조적으로 중요한" 노드(too-big-to-fail 후보, 핵심 중개기관) 식별. 철도 사례의 교훈처럼 "거래량 큰 종목 ≠ 구조적으로 중요한 종목"일 수 있으므로 가중치(거래량·시가총액)를 관계 속성으로 넣어 betweenness를 재계산해야 한다. (3) Community detection → 동조화되어 움직이는 종목 클러스터/테마 발굴. (4) Similarity → 유사 비즈니스 모델·재무구조 기업 군집. (5) Link prediction → 아직 명시되지 않은 공급/경쟁 관계 추론. 모델링 시 중심성·군집 결과를 `.write`로 노드에 되써 팩터/피처로 운영화하는 것이 인사이트 도출의 핵심 루프다.

## Summary (핵심 정리)
- GDS는 프로젝션 기반 4단계로 위상에서 인사이트를 추출하며, Python/Cypher로 65+ 알고리즘을 비전문가도 실험·운영화할 수 있다.
- 결과는 반직관적일 수 있으므로 가중치·프로젝션·가설을 반복 정제해야 하며, 운영에서는 primary/secondary 분리와 .write 보강이 정석이다.
- 주식시장에서는 전파·중심성·커뮤니티·유사성·링크예측이 시스템 리스크·핵심 노드·테마 클러스터·숨은 관계를 정량화하는 강력한 도구이며, 가중치 설계가 결과 타당성을 좌우한다.
