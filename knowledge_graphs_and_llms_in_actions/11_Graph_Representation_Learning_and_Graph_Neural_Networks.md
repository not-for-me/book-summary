# 11. Graph Representation Learning and Graph Neural Networks

## 챕터 개요 (3줄 요약)

- 그래프 표현 학습(GRL, Graph Representation Learning)은 수동 피처 엔지니어링의 확장성 한계를 극복하고 딥러닝으로 임베딩을 자동 학습한다.
- 임베딩은 인코더-디코더(encoder-decoder) 프레임워크로 통합 이해되며, 얕은 임베딩(shallow)부터 지식 그래프 임베딩, GNN(Graph Neural Network)으로 발전한다.
- GNN은 메시지 패싱(message passing)으로 이웃 정보를 반복 집계·갱신하며, 정규화·어텐션·skip connection 등으로 강화되고 LLM과 상호 보완한다.

---

## 1. Embeddings in Graph Representation Learning

> GRL은 전통적 차원 축소→word2vec 영감(Node2Vec)→딥러닝(GNN)의 3세대로 진화했다.

- 임베딩은 이산적 그래프 구조를 연속 벡터 표현으로 변환하여(도시 지도 비유) 거리 계산·관계 이해를 쉽게 한다.
- 기하 공간: 대부분 유클리드 공간이나, 계층적 구조(조직·생물 분류·의료 용어)는 공간이 지수적으로 증가하는 쌍곡(hyperbolic) 공간이 더 충실하다(기하 딥러닝 GDL).
- 위치 임베딩(positional): 전역 구조·절대 위치 보존(링크 예측·클러스터링), 구조 임베딩(structural): 상대적 위치·로컬 패턴 포착(노드 분류, GNN이 강함).
- 전이적(transductive) 학습: 고정 노드 집합 임베딩(정적 그래프), 귀납적(inductive) 학습: 미관측 노드에도 적용 가능한 파라미터 매핑(동적 그래프).
- 지도(supervised)는 라벨로 학습을 안내하고, 비지도(unsupervised)는 그래프 구조만으로 패턴을 발견한다.

---

## 2. The Encoder–Decoder Model

> 인코더-디코더 모델은 그래프 임베딩 방법을 번역 시스템처럼 통합 이해하게 한다.

- 인코더: 인접 행렬·노드 피처를 받아 노드별 밀집 벡터(임베딩)로 변환(단순 lookup table부터 신경망까지).
- 디코더: 임베딩으로 원본 그래프 속성(노드 연결 여부, 이웃 중첩, 노드 라벨)을 재구성하며, 이 재구성 차이 최소화로 인코더를 최적화한다.
- 행렬 분해·랜덤워크·GNN 모두 이 encode-decode 패턴의 다른 구현으로 통합 이해된다.
- Node2Vec 예: 인코더가 랜덤워크로 노드를 벡터화(폭넓게/깊게 탐색 균형), 디코더가 softmax로 친구 가능성을 예측 — Karate Club에서 커뮤니티·다리 멤버·리더 역할을 포착한다.

```
  Node c --[Encoder]--> embedding z_c --[Decoder]--> reconstruct
                                                     local neighborhood
  minimize difference(decoder prediction, actual graph property)
```

---

## 3. Shallow Embeddings

> 얕은 임베딩은 인코더-디코더의 가장 단순한 구현으로, 인코더가 노드→벡터의 lookup table 역할을 한다.

- 인코더는 노드별 임베딩 행을 가진 행렬을 유지하고, 디코더는 이 벡터로 그래프 속성을 재구성한다.
- 한계: 파라미터 비효율(노드 수에 선형 증가), 파라미터 미공유, 피처 무시(feature blindness), 전이적(transductive) 특성(미관측 노드 임베딩 불가).
- 이 한계가 구조·피처 기반으로 귀납 학습하는 GNN 개발을 이끌었으나, 작고 정적인 그래프·제한된 자원에선 여전히 유용하다.

---

## 4. Embeddings in Knowledge Graphs

> 실세계 KG는 다양한 관계 타입을 가진 다중관계(multirelational) 그래프로 더 정교한 임베딩이 필요하다.

- 임베딩은 엔티티가 연결되었는지뿐 아니라 어떻게(TREATS, INHIBITS, CAUSES) 연결되는지 인코딩해야 한다.

### 11.4.1 Loss Function

- 모든 노드 쌍 비교는 연산 불가능(100만 사용자→1조 쌍)하고 그래프는 희소(sparse)하므로, 네거티브 샘플링(negative sampling)+교차 엔트로피 손실을 쓴다.
- 손실은 참 사실(Aspirin-TREATS-Headache)에 높은 점수를, 무작위 샘플링한 거짓 사실에 낮은 점수를 주도록 학습하며, γ로 균형을 맞춘다(보통 γ>1).
- 네거티브 샘플링 전략: 타입 제약(type-constrained), 적대적(adversarial), 주어/목적어/둘 다 교체로 편향을 방지한다.

### 11.4.2 Multirelationship Decoder

- 대칭/비대칭 관계, 합성(compositional) 패턴, 역(inverse) 관계를 포착해야 한다.
- 번역 기반(TransE: 관계 벡터로 엔티티 이동), 행렬 기반(RESCAL: 관계=행렬, 파라미터 많음), 의미 매칭(DistMult, ComplEx: 복소수로 비대칭 처리).
- 디코더 선택이 학습 가능한 패턴을 좌우한다(TransE는 합성에 강하나 다대일에 약하고, ComplEx는 비대칭에 강함).

---

## 5. Message Passing and Graph Neural Networks

> GNN은 신경 메시지 패싱(neural message passing)으로 노드가 이웃에서 반복 학습하게 한다.

- 메시지 패싱은 라운드마다 노드가 이웃 메시지 수집→처리→자기 표현 갱신을 수행하며, AGGREGATE(이웃 메시지 집계)와 UPDATE(노드 표현 갱신) 함수로 형식화된다.
- k번 반복 후 각 노드 표현은 k홉 이웃 정보를 담아 구조적·피처 기반 정보를 포착한다.
- 기본 GNN은 학습 가능 행렬(W_self, W_neigh)과 비선형 활성화(ReLU/tanh)로 갱신하며, self-loop 변형은 AGGREGATE·UPDATE를 합쳐 단순화한다.

```
  for k iterations:
    h_u^(k) = sigma( W_self * h_u + W_neigh * SUM(h_v for v in N(u)) + b )
  (each node gathers info from k-hop neighborhood)
```

---

## 6. Generalized Aggregation and Update Methods

> 메시지 패싱은 정규화·어텐션·갱신 방법으로 강화될 수 있다.

- 아키텍처 강화: skip connection(over-smoothing 방지, 깊은 GNN 가능), 어텐션(이웃 선택적 집계), 정규화(다양한 이웃 크기 안정화).
- 이웃 정규화: 평균 정규화(합 대신 평균), Kipf-Welling의 대칭 정규화(소스·타깃 차수 모두 고려, 고인용 논문 영향 감소).
- 이웃 어텐션: GAT(Graph Attention Network)가 학습 가중치로 이웃 중요도를 다르게 부여하며, GraphSAGE도 집계에 어텐션을 통합한다.
- 멀티헤드 어텐션·트랜스포머 연결: 여러 독립 어텐션 헤드가 병렬로 다른 관계 측면을 학습하며, Q·K·V 행렬과 위치/구조 인코딩으로 트랜스포머와 수렴한다.
- 일반화된 갱신: skip connection(정보 보존), gated update(RNN 영감, 선택적 이웃 정보 통합), jumping knowledge network(여러 계층 표현을 LSTM으로 적응적 결합).

---

## 7. The Synergy of GNNs and LLMs

> GNN과 LLM은 상호 보완적 강점을 결합할 때 더 강력해진다.

- GNN은 구조 그래프 처리(메시지 패싱, 노드 분류·링크 예측)에 강하나 풍부한 텍스트 처리에 약하고, LLM은 자연어 이해·생성에 강하나 구조 관계에 약하다.
- LLM as predictors: 그래프 구조를 시퀀스로 인코딩하거나 LLM이 GNN 임베딩+질문으로 자연어 답변 생성(KG 기반 QA).
- LLM as encoders: LLM이 노드·엣지 텍스트(논문 초록)를 피처 벡터로 인코딩하고 GNN이 구조 처리.
- LLM as aligners: 대조 학습(contrastive learning)으로 LLM·GNN 출력을 병렬·정렬(멀티모달 KG).

---

## Summary (핵심 정리)

- GRL은 노드·엣지를 밀집 벡터로 변환해 피처 엔지니어링을 자동화하며, 전통 차원 축소→word2vec 영감→GNN의 3세대로 진화했고 위치/구조·전이/귀납·유클리드/비유클리드 선택이 중요하다.
- 인코더-디코더 프레임워크가 다양한 임베딩 접근을 통합하며, 얕은 임베딩은 단순 기준선, KG 임베딩은 네거티브 샘플링 손실과 다중관계 디코더(TransE·RESCAL·ComplEx)를 쓴다.
- GNN의 메시지 패싱은 이웃 정보 반복 집계로 피처를 자동 발견하며, 멀티헤드 어텐션·정규화·skip connection·gated update로 강화되고, LLM은 predictor·encoder·aligner로 GNN을 보완한다.
