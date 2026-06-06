# 12. Node Classification and Link Prediction with GNNs

## 챕터 개요 (3줄 요약)

- GNN을 노드 분류(자금세탁방지 AML)와 링크 예측(영화 추천)이라는 두 실세계 과제에 적용하며, 두 시나리오 모두 동일한 인코더-디코더(encoder-decoder) 프레임워크로 다룬다.
- PyTorch Geometric(PyG)으로 GCN(Graph Convolutional Network)·GraphSAGE(SAGE)·GAT(Graph Attention Network) 세 아키텍처를 구현·비교하고 precision·recall·F1-score·혼동 행렬로 평가한다.
- 두 과제 모두에서 SAGE가 정밀도와 균형(F1)에서 우수하며, GNN이 사기 탐지·추천 등 다양한 도메인에 가치 있음을 보인다.

---

## 1. Node Classification for Anti-Money Laundering Applications

> 금융 거래를 그래프(노드=계정, 엣지=거래)로 모델링하여 합법/불법 노드를 탐지한다.

### 12.1.1 ~ 12.1.2 Input Data & Data Preparation

- Elliptic 데이터셋: 20만+ 비트코인 거래(노드), 23만+ 결제 흐름(엣지), 166개 익명 피처, 세 CSV(features·edgelist·classes)로 제공.
- 클래스 분포: unknown 77.15%, 합법(licit) 20.62%, 불법(illicit) 2.23%로 불균형하다.
- 전처리: 노드에 증분 ID 부여, edge_index 텐서 생성, node_features 텐서([203769, 166]) 생성, LabelEncoder로 라벨을 수치화.

### 12.1.3 Homogeneous PyG Graph

- node_features·edge_index·node_labels로 PyG Data 객체를 만들고, 알려진 라벨(known_mask)만 모델에 보이게 마스킹한다.
- 학습(80%)·검증(10%)·테스트(10%) 마스크로 분할하며 각 데이터셋의 합법/불법 균형을 확인한다.

### 12.1.4 Encoder–Decoder Architecture

- NodeClassifier: GNN 인코더(이웃 정보로 노드 표현 갱신)+log_softmax 디코더(노드별 확률).
- BaseGraphModel은 2계층 아키텍처로 어떤 PyG GNN(SAGEConv 등)도 합성곱(convolution)에 쓸 수 있게 한다.
- SAGE는 추가 파라미터 불필요해 단순하고, GAT는 GATConv의 멀티헤드 어텐션 파라미터(num_heads 등)로 base 클래스를 확장한다.
- 디코더: log_softmax(수치 안정성)가 CrossEntropyLoss와 함께 작동해 예측과 실제 라벨 차이를 측정·최적화한다.

```
  Transaction Graph --[GNN Encoder: GCN/GAT/SAGE]--> node embeddings
                    --[log_softmax decoder + CrossEntropyLoss]--> licit/illicit
```

### 12.1.5 Evaluation and Analysis

- 파라미터·학습시간: GCN(2,723개, 19초)이 가장 효율적, GAT(22,025개, 43초)가 어텐션으로 가장 비효율, SAGE가 중간.
- precision·recall·F1: SAGE가 정밀도·F1 최고(거짓 양성 최소), GAT가 근접, GCN은 정밀도 최저(특히 초기 epoch).
- 혼동 행렬: SAGE가 불법 노드 약 83%·합법 약 99% 정확 분류로 최고, GAT 약 81%, GCN 약 68%(불법 노드 1/3 오분류).
- 결론: SAGE가 균형·정확도·낮은 오분류로 AML 같은 불법 노드 탐지가 중요한 응용에 가장 적합하다.

---

## 2. Link Prediction for Movie Recommendations

> 사용자-영화 상호작용을 링크로 모델링하여 관련 영화를 추천하고 무관한 추천을 피한다.

### 12.2.1 ~ 12.2.2 Input Data & Preparation

- MovieLens 소형 데이터셋: 10만 평점, 9천 영화, 600 사용자(movies.csv의 movieId·genres, ratings.csv의 userId·movieId 사용).
- 장르를 원-핫(get_dummies)으로 [9742, 20] 피처 벡터로 변환하고, 사용자·영화 ID를 증분 매핑해 edge_index([2, 100836]) 생성.

### 12.2.3 Heterogeneous PyG Graph

- 노드 타입이 둘(user, movie)이라 HeteroData를 쓰며, 단일 타입 가정의 Data와 달리 타입별 피처·관계(rates)를 구분한다.
- ToUndirected로 역방향 엣지를 추가해 user↔movie 메시지 패싱을 명시한다.
- RandomLinkSplit으로 학습(80%)·검증(10%)·테스트(10%) 엣지를 겹침 없이 분할하며, disjoint_train_ratio로 메시지 패싱용(70%)과 지도(supervision)용(30%) 엣지를 나누고 네거티브 샘플(neg_sampling_ratio)을 생성한다.
- LinkNeighborLoader로 미니배치 서브그래프를 샘플링(num_neighbors=[20,10])하여 대규모 그래프 확장성을 확보한다.

### 12.2.4 Encoder–Decoder Architecture

- MovieLensLinkPredictor: 임베딩 생성+이종(heterogeneous) GNN 인코더+DotProduct 디코더.
- 임베딩: 사용자는 고유 피처가 없어 임베딩 행렬에서 학습, 영화는 장르 피처에 선형 변환+임베딩 레이어를 결합(2단계).
- 이종 GNN: PyG의 to_hetero()로 동종 base GNN을 이종 모델로 자동 변환하며 metadata로 어떤 엣지 타입에 합성곱을 적용할지 지정한다.
- 디코더: user·movie 임베딩의 내적(dot product)으로 호환성 점수를 계산하고, binary_cross_entropy_with_logits(sigmoid+BCE)로 링크 존재 확률을 학습한다.

```
  user --rates--> movie  (HeteroData, + rev_rates reverse edge)
  embeddings --[Hetero GNN: H-GraphConv/H-GAT/H-SAGE]--> node reps
  --[dot product decoder + BCE]--> P(link exists)
```

### 12.2.5 Evaluation and Analysis

- 파라미터·학습시간: SAGE(713,408개, 777초) 최효율, GAT(1,066,880개, 956초) 최비효율, GCN 중간(노드 분류보다 파라미터·시간이 훨씬 큼 — 임베딩 레이어·이종 모델 때문).
- precision: SAGE 최고(무관 추천 최소), GCN 근접, GAT 최저·변동 큼. recall: GCN 최고(가장 포괄적), SAGE 근접, GAT 부진.
- F1: SAGE가 정확성과 커버리지 균형 최고, GCN은 높은 recall로 양호하나 정밀도가 약함, GAT는 과추천 경향.
- 혼동 행렬: SAGE가 거짓 양성 최소(5.4%)로 정밀, GCN이 잠재 평점 포착과 무관 추천 회피의 최선 균형, GAT는 사용자가 평가 안 할 영화를 과추천한다.

---

## Summary (핵심 정리)

- GNN은 노드 분류·링크 예측 같은 그래프 ML의 근본 과제를 해결하며, 과제 도메인이 달라도 인코더(GCN·GAT·SAGE)-디코더(과제별 함수) 프레임워크로 일반화할 수 있다.
- 인코더는 GNN 아키텍처, 디코더는 학습된 표현에 과제 특화 함수(노드 분류는 log_softmax, 링크 예측은 dot product)를 적용한다.
- GNN의 복잡한 관계 포착 능력은 사기 탐지·추천 시스템 등 실세계 문제와 다양한 도메인에 매우 가치 있다.
