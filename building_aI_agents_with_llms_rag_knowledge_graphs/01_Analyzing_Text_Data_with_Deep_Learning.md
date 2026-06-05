# 01. Analyzing Text Data with Deep Learning

## 챕터 개요 (3줄 요약)

- 텍스트를 기계가 이해할 수 있는 수치 벡터로 변환하는 방법(one-hot, BoW, TF-IDF, word2vec)을 단계적으로 다룬다.
- 단어의 의미와 문맥을 보존하는 밀집(dense) 임베딩과 유사도 계산(코사인 유사도) 개념을 설명한다.
- 순차 데이터를 처리하는 딥러닝 모델(RNN, LSTM, GRU, CNN)을 소개하고 감정 분석에 적용한다.

---

## 1. Representing text for AI

> 텍스트는 기호와 의미 사이에 고유한 관계가 없어 수치로 표현하기 가장 어려운 데이터이며, 정규화와 토큰화를 거쳐 벡터로 변환해야 한다.

- 말뭉치(corpus)를 기본 단위인 단어로 나누는 과정을 텍스트 정규화(text normalization)라 한다.
- 대소문자 통일, 표제어 추출(lemmatization), 어간 추출(stemming) 등으로 같은 의미의 단어를 통합한다.
- One-hot encoding은 어휘 크기만큼의 차원에서 한 위치만 1인 희소 벡터를 만든다.
- One-hot의 단점: 의미 관계를 잃고, 어휘가 약 20만 단어로 커지면 매우 고차원·희소해진다.
- BoW(Bag of Words)는 단어 순서를 무시하고 빈도(frequency) 정보만 보존한다.
- BoW도 어휘가 커지면 벡터가 커지고 희소해지는 차원의 저주(curse of dimensionality) 문제가 있다.
- TF-IDF(Term Frequency-Inverse Document Frequency)는 빈도를 로그로 정규화하고 일부 문서에만 등장하는 단어에 가중치를 준다.
- TF-IDF는 "good", "bad"처럼 흔한 단어의 영향을 줄이고 변별력 있는 단어를 강조한다.

### One-hot / BoW / TF-IDF 비교

```
text -> normalization -> tokenization -> vectorization
  one-hot : sparse, presence/absence only
  BoW     : sparse, word frequency
  TF-IDF  : TF * log(IDF), rare-but-informative words weighted
```

---

## 2. Embedding, application, and representation

> 분포 가설(distributional hypothesis)에 따라 비슷한 문맥에 등장하는 단어는 비슷한 의미를 가지며, 이를 밀집 벡터(word embedding)로 표현한다.

- 임베딩은 크기가 작고 실수로 구성된 밀집 벡터로, 어휘 크기가 늘어도 차원이 커지지 않는다.
- word2vec(2013, Mikolov)은 문맥으로부터 단어를 예측하도록 신경망을 학습하고, 그 가중치를 임베딩으로 사용한다.
- word2vec은 "단어 c가 단어 w의 문맥에 있는가?"라는 이진 분류와 로지스틱 회귀로 단순화된다.
- 컨텍스트 윈도우(context window)를 슬라이딩하며 긍정 예시를 모으고, 무작위 단어를 부정 예시로 샘플링한다.
- 임베딩 품질은 데이터 품질, 데이터 양, 차원 수(약 300이 최적), 윈도우 크기에 영향을 받는다.
- 유사도는 주로 내적(dot product) 기반이나, 빈도·크기에 민감해 코사인 유사도(cosine similarity)를 선호한다.
- 코사인 유사도는 -1~1 범위로 해석이 쉽고 스케일 불변(scale-invariant)이며 고차원에도 적합하다.
- 임베딩은 유추(analogy, king:queen::man:?), 시대별 의미 변화, 단어 다의성(여러 의미의 선형 중첩)도 인코딩한다.

### word2vec 학습 구조

```
[ ... context window ... ]   center word w
   c1  c2  [ w ]  c3  c4   -> positive pairs
   random words            -> negative samples
   train classifier -> weights = embeddings
```

---

## 3. RNNs, LSTMs, GRUs, and CNNs for text

> 텍스트의 순차적 특성을 다루기 위해 이전 입력의 메모리를 유지하는 순환 신경망 계열 모델과 1D 합성곱 모델을 사용한다.

- RNN(Recurrent Neural Network)은 내부 은닉 상태(hidden state)로 이전 입력의 정보를 기억한다.
- RNN은 입력 길이에 제한이 없지만 순차적이라 병렬화가 어렵고, 기울기 소실(vanishing gradient) 문제가 있다.
- LSTM(Long Short-Term Memory)은 게이트(forget, input, output)와 별도 컨텍스트(c)로 정보를 선택적으로 보존한다.
- LSTM은 약 100 타임스텝까지 기억 가능하며, plus 연산으로 기울기 소실·폭발을 완화한다.
- GRU(Gated Recurrent Unit)는 update gate와 reset gate를 쓰는 더 가벼운 변형으로, 파라미터가 적고 수렴이 빠르다.
- GRU는 LSTM과 성능이 비슷하나 장기 의존성·복잡 패턴에서는 다소 약하고 과적합 위험이 있다.
- CNN(Convolutional Neural Network)은 1D 필터(커널)를 시퀀스에 슬라이딩해 지역 패턴을 추출한다.
- 1D 합성곱은 매우 빠르며 임베딩 위에 적용 가능하고, max pooling으로 중요한 특징을 추출한다.

### LSTM 셀 게이트 흐름

```
   x_t, h_(t-1)
       |
  +----+----+----+
  |forget|input|output|
  +----+----+----+
   c_(t-1) -> [forget] -> + [input*g] -> c_t
   c_t -> tanh * output -> h_t
```

---

## 4. Performing sentiment analysis with embedding and deep learning

> 50,000개의 영화 리뷰를 긍정/부정으로 분류하는 모델을 임베딩과 GRU를 결합해 학습한다.

- 데이터셋은 긍정·부정 리뷰 각 균형 분포이며 평균 약 230단어 길이를 가진다.
- 라벨을 이진 인코딩(positive=0, negative=1)하고 train/validation/test로 균형 분할한다.
- 전처리: 공백·특수문자·구두점 제거, 토큰화, 불용어(stopword)와 단일 문자 단어 제거, 상위 1,000 단어만 사용.
- 단어를 어휘 인덱스로 벡터화하고, 길이를 맞추기 위해 패딩(padding)을 적용한다.
- 모델은 임베딩(차원 300) + GRU(3개 층) + 드롭아웃 정규화 + 선형 층으로 구성한다.
- 손실 함수는 이진 교차 엔트로피(BCELoss), 옵티마이저는 Adam을 사용한다.
- 학습 후 혼동 행렬(confusion matrix)에서 좋은 정확도를 보이며, 긍/부정 리뷰가 임베딩 공간에서 분리된다.

---

## Summary (핵심 정리)

- 텍스트를 one-hot, BoW, TF-IDF를 거쳐 의미를 보존하는 word2vec 임베딩으로 변환하는 과정을 익혔다.
- RNN, LSTM, GRU, CNN 같은 딥러닝 모델로 순차 텍스트를 분석하는 방법을 배웠다.
- 이 기초들을 결합해 감정 분석을 수행했으며, 이는 LLM(Large Language Model) 내부 동작 이해의 토대가 된다.
