# 12. Practical Graph Data Science

## 챕터 개요 (3줄 요약)

- Neo4j 그래프 데이터 사이언스(GDS, Graph Data Science) 라이브러리가 제공하는 병렬 그래프 알고리즘과 인메모리 그래프 카탈로그(graph catalog) 개념을 소개한다.
- 플레이리스트 공동출현(co-occurrence) 그래프를 만들고 커뮤니티 탐지(community detection) 알고리즘을 적용하는 실습을 단계별(투영→메모리 추정→실행→저장→분석)로 진행한다.
- 탐지된 커뮤니티를 플레이리스트 추천, 사용자 세분화, 인플루언서 발견, 콘텐츠 라이선싱 전략 등 실제 비즈니스 사례에 적용한다.

---

## 1. Introduction to the GDS Library

> GDS 라이브러리는 Cypher 프로시저로 접근하는 고성능 병렬 그래프 알고리즘과 ML 파이프라인을 제공한다.

- 알고리즘(algorithms): 그래프/노드/관계의 지표를 계산해 핵심 엔티티(중심성·랭킹)와 구조(커뮤니티 탐지·파티셔닝·클러스터링)에 대한 통찰을 준다.
- 보통 랜덤 워크(random walk), BFS/DFS, 패턴 매칭, 그래프 임베딩(embedding), 경로 탐색 같은 반복적 기법으로 그래프를 순회한다.
- ML 파이프라인으로 누락된 관계 예측 같은 지도 학습 모델을 훈련할 수 있다.
- 그래프 카탈로그(graph catalog): 알고리즘 효율을 위해 그래프의 부분집합을 컴팩트한 인메모리 형식으로 로드하며, 그래프 투영(projection)으로 라벨·관계 타입 등으로 필터링한다.

---

## 2. AI-Driven Playlist Communities

> 커뮤니티 탐지는 네트워크 내 클러스터(그룹)를 식별하는 기본 기법으로, 플레이리스트의 숨은 패턴을 드러낸다.

- 공동출현 그래프(co-occurrence graph): 각 노드는 플레이리스트이고, 두 플레이리스트가 트랙의 20~30%를 공유하면 엣지를 그리며(초기 임계값은 가설적으로 정하고 반복 실험으로 조정), 엣지 가중치는 공유 트랙 수다.
- 작게 시작(start small): 실험 데이터셋을 `ExperimentOne` 같은 라벨로 표시하고, 낮은 공동출현은 건너뛰어 노이즈를 줄인다.
- 공동출현 관계를 생성한 뒤 GDS를 적용한다.
- 커뮤니티(community)는 서로 비슷한 취향·구조를 공유하는 플레이리스트 묶음을 의미한다.

---

## 3. Using GDS Workflow

> GDS 사용은 부분 그래프 투영 → 메모리 추정 → 알고리즘 실행 → 결과 저장 → 분석의 일관된 흐름을 따른다.

- 부분 그래프 투영(projecting the subgraph): 전체가 아니라 실험 대상(`ExperimentOne`)만 인메모리로 투영하며, 네이티브 투영(native) 또는 Cypher 투영 방식을 쓴다.
- 메모리 사용량 추정(estimating memory usage): 투영/실행 전 필요한 메모리를 추정해 안전하게 진행한다.
- 커뮤니티 탐지 실행: 실행 모드를 선택(stream/write 등)하고 결과를 그래프에 저장(storing results)한다.
- 분석 쿼리(analytical queries) 후 임계값·파라미터를 조정해 반복(rinse and repeat)하며 두 번째 실험으로 확장한다.

```
GDS workflow:
  Neo4j DB --(projection: labels/rel-types)--> in-memory graph catalog
           --> memory estimation --> run algorithm (community detection)
           --> store results back to DB --> analytical Cypher queries --> iterate
```

---

## 4. Real-World Applications of Community Detection

> 저장된 커뮤니티 구조는 Cypher와 GDS만으로 즉시 활용 가능한 여러 비즈니스 사례를 만든다.

- 플레이리스트 추천(playlist recommendations): 실시간 유사도 계산 대신 같은 커뮤니티에 속한 플레이리스트를 반환해 비용을 크게 줄인다.
- 사용자 세분화(user segmentation): 사용자가 따르는 플레이리스트의 최빈 커뮤니티로 행동 기반의 동적 세그먼트를 부여한다(정적 인구통계 세분화보다 우수).
- 인플루언서 발견(influencer discovery): 커뮤니티 내 영향력 있는 노드를 찾는다.
- 행동 클러스터(behavioral clusters)와 콘텐츠 라이선싱 전략(content licensing): 상위 커뮤니티 전반에 자주 등장하는 고영향 트랙을 식별해 라이선싱 우선순위를 정한다.

---

## Summary (핵심 정리)

- GDS는 Cypher 프로시저로 접근하는 병렬 그래프 알고리즘과 ML 파이프라인을 제공하며, 인메모리 그래프 카탈로그에 부분 투영해 효율적으로 실행한다.
- 공동출현 그래프에 커뮤니티 탐지를 적용하는 실습은 투영→메모리 추정→실행→저장→분석→반복의 표준 워크플로를 따른다.
- 탐지된 커뮤니티는 추천·세분화·인플루언서 발견·콘텐츠 라이선싱 등 즉시 적용 가능한 비즈니스 가치를 창출한다.
