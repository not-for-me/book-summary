# 02. Understanding Foundation Models

## 챕터 개요 (3줄 요약)
- foundation model 간 차이는 training data, model architecture/size, post-training(human preference alignment) 결정에서 비롯된다.
- transformer architecture와 attention mechanism, 모델 크기·scaling law, pre-training/post-training(SFT, preference finetuning)을 다룬다.
- sampling(temperature, top-k, top-p)이 AI를 확률적으로 만들며 창의성과 동시에 inconsistency·hallucination을 유발함을 설명한다.

---

## 1. Training Data
> 모델은 학습 데이터만큼만 좋으며, 데이터 분포가 모델의 능력과 한계를 결정한다.

Common Crawl, C4 같은 대규모 웹 데이터는 가용성 때문에 널리 쓰이지만 품질 문제(가짜뉴스 등)가 있다. "가진 것을 쓴다"는 접근은 원하는 작업에 맞지 않을 수 있어, 특정 언어·도메인에 맞춘 데이터 큐레이션이 필요하다.

### Multilingual Models
> 영어가 인터넷 데이터의 절반을 차지해, general-purpose 모델은 저자원(low-resource) 언어에서 성능이 떨어진다.

MMLU에서 GPT-4는 영어가 Telugu 등보다 훨씬 우수하다. 단순 번역 우회는 정보 손실 위험이 있다. 비영어는 tokenization이 비효율적이라(예: Burmese는 영어의 10배 token) 더 느리고 비싸다. 이를 위해 ChatGLM, PhoGPT 등 언어 특화 모델이 등장했다.

### Domain-Specific Models
> 일반 모델은 학습에서 못 본 도메인 특화 작업(신약 개발, 암 진단 등)에는 약하다.

protein/DNA, X-ray/fMRI 같은 데이터는 공개 웹에 드물고 비싸다. AlphaFold(단백질), BioNeMo(신약), Med-PaLM2(의료) 등이 특화 데이터로 학습된 예다.

---

## 2. Modeling
> 학습 전 architecture와 parameter 수를 결정하며, 이는 능력과 downstream 사용성에 영향을 준다.

### Transformer Architecture & Attention
> attention mechanism 기반 transformer가 RNN 기반 seq2seq의 한계를 해결하며 지배적 구조가 되었다.

seq2seq의 두 문제(최종 hidden state만 사용, 순차 처리)를 attention이 해결한다. attention은 query(Q, 현재 디코더 상태), key(K, 이전 token 식별), value(V, 실제 내용) 벡터의 dot product로 각 token에 줄 가중치를 계산한다. inference는 prefill(입력 병렬 처리)과 decode(token 순차 생성) 두 단계로 나뉜다. 보통 multi-head이며, transformer block은 attention module과 MLP(Multi-Layer Perceptron) module로 구성된다.

```
Attention(Q,K,V) = softmax(Q*K^T / sqrt(d)) * V

  K = x*Wk,  V = x*Wv,  Q = x*Wq
```

### Other Architectures
> transformer를 능가하기는 어렵지만 RWKV, SSM 계열이 부상 중이다.

RWKV(병렬화 가능 RNN), SSM(State Space Model) 계열인 S4, H3, Mamba(선형 시간 확장), Jamba(transformer-Mamba 하이브리드)가 긴 시퀀스 처리에서 가능성을 보인다.

### Model Size & Scaling Law
> parameter 수, training token 수, FLOP가 모델 규모의 세 지표다.

parameter가 많을수록 학습 용량이 크지만 충분한 데이터가 필요하다. sparse model(MoE, Mixture-of-Experts)은 일부 expert만 활성화해 효율적이다(예: Mixtral 8x7B는 token당 12.9B만 활성). compute 측정 단위는 FLOP(floating point operation)이며 FLOP/s(초당)와 구분된다. Chinchilla scaling law: compute-optimal을 위해 training token은 모델 크기의 약 20배여야 한다.

```
3 scale signals:
  #parameters  -> learning capacity
  #tokens      -> how much learned
  #FLOPs       -> training cost
```

### Scaling Bottlenecks
> training data와 electricity가 scaling의 두 병목이다.

수년 내 공개 인터넷 데이터 고갈 우려가 있고, AI 생성 데이터의 재귀 학습은 성능 저하 위험이 있다. proprietary data가 경쟁 우위가 된다. data center 전력 소비도 주요 제약이다.

---

## 3. Post-Training
> pre-trained 모델을 human preference에 맞추는 단계로, 대화 최적화와 부적절 출력 완화를 목표로 한다.

두 단계로 구성된다.

### Supervised Finetuning (SFT)
> 고품질 (prompt, response) demonstration data로 완성이 아닌 대화에 맞게 미세조정한다.

pre-trained 모델은 완성에 최적화되어 있어 demonstration data(behavior cloning)로 적절한 응답을 학습시킨다. 다양한 작업(QA, 요약, 번역)을 포함해야 하며, 고학력 labeler가 데이터를 생성해 비용이 크다(InstructGPT 13,000쌍 ~$130K).

### Preference Finetuning
> human preference에 맞게 행동하도록, 주로 RL(Reinforcement Learning)로 미세조정한다.

RLHF(Reinforcement Learning from Human Feedback)는 (1) reward model 학습 후 (2) 그 점수를 최대화하도록 최적화한다. reward model은 pointwise 대신 (prompt, winning, losing) comparison data로 학습한다. PPO(Proximal Policy Optimization)로 최적화하며, DPO(Direct Preference Optimization), RLAIF(from AI Feedback) 등 대안이 있다. best of N 전략으로 RL을 생략하기도 한다.

```
Pretrain (monster) -> SFT (socially acceptable) -> Preference FT (smiley face)
```

---

## 4. Sampling
> 모델이 가능한 출력 중 하나를 고르는 과정으로, AI를 확률적으로 만든다.

### Sampling Fundamentals & Strategies
> logit을 softmax로 확률화한 뒤 분포에 따라 다음 token을 샘플링한다.

greedy sampling(최고 확률)은 지루한 출력을 낳는다. temperature는 logit을 나눠 분포를 조정한다(높으면 창의적, 낮으면 일관적; 0.7 권장). top-k는 상위 k개 logit만 고려해 계산을 줄이고, top-p(nucleus)는 누적 확률 p까지 동적으로 고려한다(0.9~0.95). stopping condition으로 latency·cost를 줄이되 format 손상에 주의한다.

```
logits -> /temperature -> softmax -> probabilities -> (top-k / top-p) -> sample
```

### Test Time Compute
> 쿼리당 여러 응답을 생성해 좋은 응답 확률을 높인다.

best of N, beam search로 여러 후보를 생성하고, 출력 다양성을 높이면 효과가 커진다. 선택은 최고 average logprob, reward model/verifier 점수, 또는 다수결(self-consistency)로 한다. verifier 사용은 30배 모델 크기 증가에 맞먹는 성능 향상을 줄 수 있으나 비용이 크다.

### Structured Outputs
> production에서는 특정 형식(JSON, SQL 등) 출력이 필요하다.

semantic parsing(text-to-SQL 등)과 downstream 앱 입력용으로 구조화가 필요하다. 접근법: prompting, post-processing(흔한 오류 교정), test time compute, constrained sampling(grammar로 유효 token만 필터), finetuning(가장 효과적·일반적). classification은 classifier head 추가로 형식을 보장할 수 있다.

---

## 5. The Probabilistic Nature of AI
> sampling이 AI를 확률적으로 만들어 창의성에 유리하지만 inconsistency와 hallucination을 일으킨다.

### Inconsistency
> 같은/유사 입력에 매우 다른 출력을 내는 현상.

같은 입력-다른 출력은 caching, sampling 변수 고정, seed 고정으로 완화하지만 100% 보장은 어렵다(하드웨어 영향 포함). 약간 다른 입력-크게 다른 출력은 더 어려우며 신중한 prompt와 memory system으로 개선한다.

### Hallucination
> 사실에 근거하지 않은 응답으로, factuality 의존 작업에 치명적이다.

두 가설: (1) self-delusion — 모델이 주어진 데이터와 자신이 생성한 데이터를 구분 못해 snowballing hallucination 발생, (2) mismatched internal knowledge — SFT에서 labeler가 아는 지식을 모델이 모르는데 모방하게 되어 발생. 완화책으로 verification(출처 검색), 더 나은 reward function, 간결한 응답 유도, prompt 기법이 있다.

---

## Summary (핵심 정리)
- training data(특히 언어·도메인 큐레이션), architecture(transformer/attention), model size(parameter·token·FLOP, Chinchilla scaling law)가 핵심 설계 결정이다.
- pre-training의 한계(완성 최적화, 저품질)는 post-training(SFT + preference finetuning)으로 보완하나 human preference를 완벽히 담기는 어렵다.
- sampling은 AI를 확률적으로 만들어 창의성의 원천이자 inconsistency·hallucination의 원인이며, 이를 체계적으로 다루는 것이 AI engineering의 과제다.
