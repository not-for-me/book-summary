# 02. The Transformer: The Model Behind the Modern AI Revolution

## 챕터 개요 (3줄 요약)

- RNN 계열의 한계를 어텐션(attention)과 셀프 어텐션(self-attention)이 어떻게 해결했는지 설명한다.
- 트랜스포머(transformer)의 구조(멀티헤드 어텐션, FFN, 잔차 연결, 층 정규화)와 학습 방식을 다룬다.
- BERT의 마스킹 언어 모델링, 내부 메커니즘 시각화, 파인튜닝·지식 증류 응용까지 살펴본다.

---

## 1. Exploring attention and self-attention

> 어텐션은 정렬(alignment) 문제를 풀기 위해 등장했으며, 디코딩 시 입력의 어느 토큰이 중요한지를 동적으로 결정한다.

- 초기 기계 번역은 seq2seq 모델(encoder-decoder)을 사용했으나 정렬, 기울기 소실/폭발, 병렬화 불가 문제가 있었다.
- 어텐션은 인코더 은닉 상태(h)와 디코더 출력(s)의 점수를 구하고 softmax로 정규화해 가중합으로 컨텍스트 벡터를 만든다.
- 어텐션은 기울기 소실 완화, 병목 제거, 해석 가능성(interpretability), 성능 향상의 이점을 준다.
- 셀프 어텐션(self-attention)은 입력 내부에서 Query, Key, Value를 비교해 표현을 추출한다(도서관 비유: query-key-value).
- Q, K, V는 입력 X에 학습 가중치 행렬을 곱해 얻으며, softmax(Q·K^T / sqrt(d))·V로 계산한다.
- 멀티헤드 어텐션(multi-head attention)은 여러 종류의 관계를 동시에 포착하고 병렬 계산이 가능하다.
- 셀프 어텐션은 순서 정보가 없고 토큰 수 N에 대해 시간·공간이 제곱(quadratic) 비용을 가진다.

### Self-attention 계산 흐름

```
X -> Q = X*Wq, K = X*Wk, V = X*Wv
score = softmax( (Q . K^T) / sqrt(d_k) )
output = score . V        (multi-head -> concat -> linear)
```

---

## 2. Introducing the transformer model

> "Attention is All You Need"는 RNN을 완전히 제거하고 멀티헤드 셀프 어텐션 층만 쌓아 트랜스포머를 구성했다.

- 입력은 토큰화(tokenization) → 임베딩 → 위치 인코딩(positional encoding, sin/cos) 순으로 처리된다.
- 위치 인코딩은 학습 파라미터가 없으며 self-attention에 순서 정보를 더해준다.
- 트랜스포머 블록은 멀티헤드 셀프 어텐션, FFN(Feedforward Network), 잔차 연결(residual connection), 층 정규화(layer normalization)로 구성된다.
- FFN은 두 선형 층과 ReLU로 비선형성을 추가하며 위치별로 분리되어 병렬화가 쉽다.
- 잔차 연결은 기울기 전달을 돕고 손실 표면(loss surface)을 매끄럽게 만든다.
- 층 정규화는 평균·표준편차로 은닉 값을 정규화해 학습을 안정화한다.
- 인코더-디코더 구조에서 디코더는 크로스 어텐션(cross-attention)과 미래를 가리는 마스킹 어텐션(masked attention)을 사용한다.
- 오늘날 생성 AI는 대부분 디코더 전용(decoder-only) 구조를 사용한다.

### Transformer 블록 구조

```
input X (n x d)
  |-> Multi-Head Self-Attention -> + (residual) -> LayerNorm
  |-> Feed-Forward Network       -> + (residual) -> LayerNorm
output (n x d)   [repeat for N blocks]
```

---

## 3. Training a transformer

> 트랜스포머는 대량의 비주석 텍스트로 자기지도(self-supervised) 언어 모델링을 통해 스스로 관계를 학습한다.

- 언어 모델링은 이전 단어들로부터 다음 단어의 조건부 확률 P(w|h)를 확률의 연쇄 법칙으로 분해한다.
- 트랜스포머 블록 뒤의 언임베더(unembedder)와 softmax가 어휘 크기 V의 로짓(logit) 벡터를 생성한다.
- 학습은 교차 엔트로피(cross-entropy) 손실로 예측 분포와 실제(one-hot) 분포의 차이를 최소화한다.
- 토큰별 반복 학습 방식을 교사 강요(teacher forcing)라 하며, 손실은 시퀀스 전체 평균이다.
- 디코딩(생성): 그리디 디코딩은 반복적이라 잘 안 쓰고, top-k, top-p, temperature 샘플링으로 품질·다양성을 조절한다.
- 미등록 단어(<UNK>) 문제 해결을 위해 서브워드(subword) 토큰화를 사용한다.
- BPE(Byte-Pair Encoding)는 자주 함께 등장하는 문자/기호를 반복 병합해 N개의 토큰 어휘를 만든다.

---

## 4. Exploring masked language modeling

> BERT(Bidirectional Encoder Representations from Transformers)는 양방향 문맥을 보는 인코더 전용 모델로 트랜스포머를 대중화했다.

- 기존 트랜스포머는 좌→우(causal)라 엔티티 오른쪽 문맥을 활용하지 못하나, BERT는 전체 시퀀스 관계를 본다.
- MLM(Masked Language Model)은 토큰의 15%를 <MASK>로 가리고 나머지 문맥으로 예측하게 학습한다.
- 특수 토큰 [CLS](입력 시작)와 [SEP](문장 구분)를 사용한다.
- BERT-BASE(12층, d=768, 12헤드, 1.1억 파라미터)와 BERT-LARGE(24층, d=1024, 24헤드, 3.4억 파라미터)가 있다.
- MLM은 15% 토큰만 학습에 쓰여 비효율적이지만, 입력을 손상시켜 복원하는 유연한 접근이다.
- 다음 문장 예측(next sentence prediction)으로 문장 쌍 관계도 학습할 수 있다.
- 2024년 연구로 BERT류 모델도 [MASK] 시퀀스를 활용해 텍스트 생성이 가능함이 밝혀졌다.

---

## 5. Visualizing internal mechanisms

> 어텐션 가중치와 뉴런 활성화를 시각화하면 각 헤드·층이 학습한 다양한 관계를 해석할 수 있다.

- BERTviz 패키지로 어텐션 헤드 간 단어 관계를 시각화하며, 진한 색은 큰 가중치를 의미한다.
- 12층 × 12헤드 = 144개 어텐션 헤드가 같은 문장에 대해 100개 이상의 표현을 만든다.
- GPT-2(Generative Pre-Trained Transformer 2)는 약 40GB 텍스트로 학습한 causal 디코더 전용 모델이다.
- Gradient X input 기법으로 다음 토큰 생성에 가장 중요한 입력 토큰을 식별한다.
- FFN(Feedforward Neural Network)은 블록 파라미터의 약 66%를 차지하며 뉴런 발화 분석 대상이 된다.
- NMF(Non-Negative Matrix Factorization)로 뉴런 활성화 차원을 줄여 문법·의미 구조 특화를 관찰한다.
- 층별로 서로 다른 표현을 학습하며, 위치 인코딩 덕분에 단어 순서도 활성화에 반영된다.

---

## 6. Applying a transformer

> 사전 학습(pre-training)으로 일반 언어 규칙을 익힌 모델을 전이 학습(transfer learning)·파인튜닝으로 특정 작업에 적응시킨다.

- 파인튜닝(fine-tuning)은 대부분 층을 동결(freeze)하고 상단에 추가한 1~2개 층만 학습한다.
- BERT는 [CLS] 토큰의 최종 벡터를 softmax에 넣어 분류 작업에 활용한다.
- 매우 낮은 학습률로 전체 가중치를 미세 조정하면 성능이 더 향상될 수 있다.
- Hugging Face의 Trainer와 TrainingArguments로 distill-BERT 등을 간단히 파인튜닝한다.
- 디코더 전용 모델은 특수 토큰(<to-fr> 등)으로 데이터셋을 구성해 다음 토큰 예측으로 번역·요약을 학습한다.
- 지식 증류(knowledge distillation)는 큰 교사(teacher) 모델의 지식을 작은 학생(student) 모델로 전이한다.
- 증류 손실은 보통 KL 발산(Kullback-Leibler divergence)으로 교사와 학생의 로짓 분포 차이를 최소화한다.

### 지식 증류 프레임워크

```
teacher (large, frozen) --logits--> 
                                    KL-divergence loss
student (small, train)  --logits-->
```

---

## Summary (핵심 정리)

- 트랜스포머는 self-attention, 임베딩, 토큰화가 결합해 NLP(Natural Language Processing)를 혁신한 모델임을 배웠다.
- 내부 메커니즘 시각화와 파인튜닝·지식 증류로 모델을 다양한 작업에 적응시키는 법을 익혔다.
- LLM(Large Language Model)은 더 많은 파라미터와 텍스트로 학습한 트랜스포머이며, 다음 장에서 그 확장을 다룬다.
