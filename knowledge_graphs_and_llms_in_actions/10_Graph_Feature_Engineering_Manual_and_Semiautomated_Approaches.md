# 10. Graph Feature Engineering: Manual and Semiautomated Approaches

## 챕터 개요 (3줄 요약)

- 그래프 ML 성공의 핵심은 노드·관계·그래프를 ML 알고리즘이 처리할 수 있는 벡터로 표현하는 벡터화(featurization)이며, 수동→반자동으로 진행한다.
- 수동 피처는 해석가능하나 노동집약적이고, 반자동(ReFeX)은 해석가능성과 효율의 균형을, 완전자동(GNN, 다음 장)은 효율적이나 해석이 어렵다.
- 사기 탐지(노드 분류)와 신약 재창출(링크 예측)을 사례로, 도메인 지식이 단순 지표보다 효과적인 피처를 만듦을 보인다.

---

## 1. Manual Node Features

> 사기 탐지 네트워크에서 각 노드를 지표·그래프 알고리즘으로 수치 피처 벡터로 변환한다.

- 로컬 피처(local): 1홉 이웃 또는 에고넷(egonet, 노드와 직접 이웃)에서 추출(차수, 삼각형, 밀도).
- 글로벌 피처(global): 전체 네트워크에서 노드 역할 측정(매개·근접·고유벡터 중심성, PageRank).

### 차수·삼각형·밀도 (로컬)

- 차수(degree): 이웃 수를 전체·사기(fraud)·정상(legit) 차수로 나눠 직접 연결을 더 잘 표현한다.
- 삼각형(triangle): 세 노드가 모두 연결된 부분그래프로 긴밀한 연결을 나타내며, 두 alter의 사기 여부로 fraud/legit/semifraud로 분류한다.
- 밀도(density): 에고넷에서 가능한 엣지 중 실제 엣지 비율(0~1)로 노드 간 상호 영향 정도를 측정한다.

### 경로·중심성 (글로벌)

- 측지 경로(geodesic/shortest path): 최단 거리로, 사기 노드까지의 거리와 1·2·3홉 경로 수를 도메인에 맞춰 커스터마이징하며 Dijkstra 알고리즘을 사용한다.
- 근접 중심성(closeness centrality): 다른 모든 노드와 얼마나 가까운지(farness의 역수)로, 낮으면 사기가 빠르게 퍼질 수 있다.
- 매개 중심성(betweenness centrality): 노드가 다른 쌍의 최단 경로에 얼마나 자주 등장하는지로 정보 흐름 통제력을 측정(노드 A가 최고).
- PageRank: 연결의 양뿐 아니라 질(고순위 노드와의 연결)을 고려하며, 사기 가중(fraud-weighted) PageRank로 사기 활동과의 관계를 본다(노드 D가 최고).

### 10.1.8 Prediction

- 추출된 모든 피처로 노드별 벡터를 만들어 pandas DataFrame으로 구성하고, 계층 분할(stratified split)·StandardScaler·로지스틱 회귀로 사기 분류기를 학습·평가한다.
- 이 접근의 장점은 과정이 완전히 통제 가능하고 각 피처를 그래프만 봐도 설명할 수 있어 투명성이 높다는 것(소규모 DB·설명가능성 중요 시 유효).

---

## 2. Manual Relationship Features

> 링크 예측(link prediction)은 노드 쌍을 입력으로 두 노드 간 관계 존재(또는 타입) 가능성을 예측한다.

- 노드 기반 결합(node-based): 소스·타깃 노드의 피처 벡터를 결합(concatenate, average, L1 맨해튼 거리, L2 유클리드 거리, Hadamard 원소곱).
- 경로 기반 피처(path-based): 노드 연결 방식(메타패스, 2홉 경로 수 등)으로 관계의 구조적 맥락을 포착하며 도메인 특화 수동 작업이 필요하다.

### 10.2.2 Path-based Features (Hetionet 사례)

- Himmelstein et al.은 Hetionet으로 화합물(compound)→질병(disease)의 길이 2~4 메타패스를 평가해 신약 재창출(우울증·알코올중독 약→금연·간질 치료 가능성)을 발견했다.
- 단순 경로 수(PC)는 고연결 노드가 지배해 오해를 부르므로, 차수 가중 경로 수(DWPC, Degree-Weighted Path Count)로 중간 노드 차수에 반비례 가중을 적용해 허브 노드 영향을 줄인다.
- 예: metformin-type 2 diabetes의 CbGaD 메타패스 DWPC=0.0007.
- 1,026개 메타패스를 통계적으로 709개로 줄이고, 도메인 지식·차수 확률 분석으로 유망한 화합물-질병 쌍을 선택해 복잡성을 낮춘다.
- LLM은 메타패스 설명을 최적화된 Cypher 질의로 변환(query generation), 패턴 제안(feature engineering), 인프라 코드 생성(code generation)을 지원한다.

```
  Relationship rep = combine(node_u_vector, node_v_vector)
     concatenate / average / L1 / L2 / Hadamard
  OR path-based: metapath DWPC features (CbGaD, CdGuD, CrCtD, ...)
```

---

## 3. Semiautomated Feature Extraction (ReFeX)

> ReFeX(Recursive Feature eXtraction)는 수동 피처 엔지니어링과 신경망 접근의 중간 지점으로, 구조적 피처를 자동·투명하게 추출한다.

- 장점: 효율성(재귀적 구조 피처 자동 추출), 일관성, 해석가능성(명확한 구조적 의미), 확장성, 그래프 간 비교 가능성.
- 두 원칙: 구조적(structural, 노드·링크 속성 불필요), 효과적(effective, 노드 속성 예측에 도움·그래프 간 전이 가능).
- 세 단계: 로컬 피처 추출(차수, 방향 그래프는 in/out-degree), 에고넷 피처 검사(에고넷 엣지 수), 재귀 피처 추출(이웃 피처를 sum/mean으로 집계).
- 가지치기(pruning): 상관 분석(고상관 쌍 제거), 로그 비닝(logarithmic binning), 임계값 기반 가지치기로 기하급수적 피처 증가를 관리한다.

### 10.3.1 ~ 10.3.2 ReFeX 수동·자동

- 노드 A 예: 1차 반복에서 이웃 차수 합(17)·평균(2.83), 2차 반복에서 이웃 합의 합(69)·평균(11.5)을 계산해 "행동(behavioral)" 정보(누구와 연결되는지)를 포착한다.
- ReFeX는 결정적(deterministic)이라 동일 입력→동일 출력으로 재현성이 높고, 그래프 변경 시 영향받는 피처만 선택적 재계산 가능해 동적 그래프에 적합하다.
- 한계: 구조 피처에만 의존해 노드 속성·엣지 타입을 직접 통합 못 하고, 최적 피처 선택에 인간 감독이 필요할 수 있다.

---

## Summary (핵심 정리)

- 수동·반자동 피처 엔지니어링은 그래프 ML의 기반으로 해석가능성과 자동화의 균형을 이루며, 로컬 지표와 글로벌 측정을 결합해 의미 있는 노드 표현을 만든다.
- 관계 표현은 노드 기반 결합(concatenation·averaging·L1/L2·Hadamard) 또는 경로 기반 피처(메타패스, DWPC)로 접근하며, 도메인 전문성이 지표 선택·검증을 이끈다.
- ReFeX 같은 반자동 접근은 해석가능한 피처를 자동 생성하면서 도메인 지식 통합 옵션을 보존하며, 수동/반자동 선택은 해석가능성·연산 자원·도메인 전문성에 달려 있다.
