# 7. Graph-Native Machine Learning

## 챕터 개요 (3줄 요약)
- 그래프와 ML의 결합은 두 방향이다: 그래프 자체를 진화시키는 in-graph ML(누락 관계·레이블·속성 예측)과 그래프에서 피처를 뽑아 외부 모델을 만드는 graph feature engineering.
- 그래프는 데이터뿐 아니라 "위상(topology)"을 가지므로, 노드 임베딩(Node2Vec, FastRP)으로 위상을 수치 피처로 인코딩하면 더 많고 질 높은 피처를 얻을 수 있다.
- Neo4j GDS는 링크 예측 ML 파이프라인을 "구현"이 아니라 "선언"으로 만들게 해주며, 결과를 그래프에 되써 모델→그래프→모델 피드백 루프를 형성한다.

## Machine Learning in a Nutshell
> ML은 "데이터로부터 프로그램을 도출"하는 것으로, 입력+과거 출력을 상관시켜 규칙(함수)을 학습한다(전통 SW의 책임을 역전).

- 모델 종류: 통계적 회귀, 신경망(딥러닝), 그래프의 다차원 위상으로 확장한 기하학적 학습(geometric learning).
- 그래프 통합 두 방식: in-graph ML(그래프 진화 예측), graph feature engineering(외부 예측모델용 피처 추출). 둘 다 피드백 루프로 그래프를 풍부하게 함.

## Topological Machine Learning
> in-graph ML은 그래프가 "자기 자신을 계산"해 누락된 관계·레이블·속성을 보강하는 기법이다.

- 예: 영화 그래프에서 Keanu Reeves-The Matrix 관계 삭제 후, Preferential Attachment(Barabási-Albert) 링크 예측으로 복원 추천(score 28.0).
- 단일 알고리즘 점수는 결정적이지 않으므로, 다른 알고리즘·더 큰 그래프·ML 파이프라인으로 개선 가능.

## Graph-Native ML Pipelines & Feature Engineering
> 그래프 피처 엔지니어링은 위상까지 피처로 활용해 전통적 컬럼 피처보다 풍부한 입력을 만든다.

- 피처 생성: PageRank/커뮤니티 같은 알고리즘 결과를 속성으로, 또는 노드 임베딩으로 위상을 직접 수치 인코딩.
- 링크 예측 파이프라인 9단계: 프로젝션 생성 → 파이프라인 선언 → 노드 속성(임베딩) 추가 → 링크 피처(combiner) 추가 → test/train/feature 분할 → 모델 후보 추가 → 메모리 추정 → 학습+평가 → 운영 배포.
- Neo4j는 기능이 내장되어 "구현이 아닌 설정 선언"만 하면 됨.

```
// 링크 예측 파이프라인 핵심 흐름 (Cypher)
gds.graph.project(...)                              // 1. 프로젝션
gds.beta.pipeline.linkPrediction.create(...)        // 2. 파이프라인
addNodeProperty(... 'fastRP' embedding ...)         // 3. 위상 임베딩
addFeature(... 'cosine' ...)                        // 4. 링크 피처
configureSplit(...) / addLogisticRegression(...)    // 5~6. 분할/모델
train(...) -> AUCPR metric                          // 8. 학습
predict.mutate(... 'SHOULD_ACT_WITH' topN threshold)// 운영 예측
```

## Recommending Complementary Actors (사례)
> 함께 출연한 배우(ACTED_WITH)를 학습해 "함께 일하면 좋을 배우(SHOULD_ACT_WITH)"를 예측하는 전체 파이프라인 실습.

- 전처리: 같은 영화 출연자 쌍에 ACTED_WITH MERGE.
- FastRP 임베딩(256차원) → cosine 유사도 링크 피처 → testFraction 0.25 분할 → LogisticRegression/RandomForest/MLP 후보 → autoTuning → AUCPR로 학습.
- predict.mutate(topN, threshold)로 추천 관계를 프로젝션에 쓰고, stream+MERGE로 그래프에 영속화, 대칭 관계 정규화.
- 외부 도구(TensorFlow/PyTorch/scikit-learn, Vertex AI/SageMaker/Azure ML)로 확장 가능 — 다리는 "그래프에서 추출한 피처 벡터". "인공지능을 부트스트랩하려면 여전히 인간 지능이 필요하다."

> [모델링 관점 - 주식시장 도메인 적용]
> 주식시장 지식그래프에서 graph-native ML의 활용: (1) Link prediction → 아직 공시되지 않았거나 데이터에 누락된 "숨은 관계"(잠재적 공급/경쟁/지분 관계, 동조 종목 쌍) 예측. (2) 노드 임베딩(FastRP/Node2Vec) → 각 종목의 그래프 내 위치(공급망·섹터·지분 구조 상의 맥락)를 수치 피처로 인코딩해, 가격/재무 같은 전통 피처와 결합한 예측모델 구축. 이것이 "온톨로지 기반 의미있는 인사이트"의 핵심 — 단순 시계열 피처가 아니라 "구조적 위치"를 피처화. (3) 노드 분류 → 미분류 신규 종목의 섹터/리스크 등급 자동 부여. 단, 금융에서는 임베딩이 블랙박스이므로 PageRank·중심성·커뮤니티 같은 해석 가능한 피처와 병행해 설명가능성을 확보하고, 룩어헤드 편향 방지를 위해 시점 분할(time-based split)에 각별히 주의해야 한다.

## Summary (핵심 정리)
- graph-native ML은 in-graph 보강(링크/레이블/속성 예측)과 graph feature engineering(위상 피처 추출) 두 축으로, 위상 임베딩이 핵심 무기다.
- Neo4j GDS 파이프라인은 선언만으로 링크 예측 학습·예측·그래프 보강을 수행하고 피드백 루프로 그래프를 진화시킨다.
- 주식시장에서는 숨은 관계 예측과 "구조적 위치" 피처화가 차별적 인사이트의 원천이나, 설명가능성과 시점 분할(룩어헤드 방지)이 금융 적용의 필수 안전장치다.
