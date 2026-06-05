# 03. Evaluation Methodology

## 챕터 개요 (3줄 요약)
- foundation model은 open-ended·black-box 특성으로 전통 ML보다 평가가 훨씬 어렵고, 체계적 평가에 대한 투자가 부족하다.
- language modeling 지표(cross entropy, perplexity, BPC, BPB)와 exact evaluation(functional correctness, similarity 측정)을 다룬다.
- AI as a judge(AI로 AI 평가)와 comparative evaluation(모델 간 비교 순위)의 작동 방식·장점·한계를 설명한다.

---

## 1. Challenges of Evaluating Foundation Models
> 지능이 높아질수록, open-ended·black-box 특성과 빠르게 포화되는 benchmark 때문에 평가가 어렵다.

PhD 수준 답을 검증하기 어렵듯 모델이 똑똑할수록 평가가 시간·전문성을 요한다. open-ended 작업은 정답 집합을 만들 수 없고, 대부분 모델이 black box다. GLUE→SuperGLUE, MMLU→MMLU-Pro처럼 benchmark가 빠르게 포화되며, general-purpose 모델은 새 능력 발견까지 평가 범위가 넓어졌다. 평가에 대한 투자·도구는 여전히 부족하다.

---

## 2. Understanding Language Modeling Metrics
> cross entropy, perplexity, BPC, BPB는 모두 언어모델의 예측 정확도를 측정하는 변형들이다.

### Entropy & Cross Entropy
> entropy는 token당 평균 정보량을, cross entropy는 모델이 다음 token을 예측하기 어려운 정도를 측정한다.

entropy가 낮을수록 언어가 예측 가능하다. cross entropy H(P,Q) = H(P) + KL(P||Q)로, 모델이 완벽히 학습하면 cross entropy는 데이터의 entropy와 같아진다.

### BPC, BPB & Perplexity
> perplexity는 entropy/cross entropy의 지수로, 다음 token 예측 시 불확실성을 측정한다.

BPC(Bits-Per-Character), BPB(Bits-Per-Byte)는 인코딩 차이를 표준화한 변형이다. perplexity(PPL)는 낮을수록 좋다. 일반 규칙: 구조화된 데이터일수록, 작은 vocabulary일수록, 긴 context일수록 perplexity가 낮다. perplexity는 모델 능력의 proxy이자 data contamination 탐지·deduplication·이상 텍스트 탐지에 쓰인다(단, post-training·quantization 후엔 신뢰도 하락).

```
PPL(P,Q) = 2^H(P,Q)   (bit 단위)
PPL(P,Q) = e^H(P,Q)   (nat 단위)
```

---

## 3. Exact Evaluation
> 모호함 없는 판정을 내는 평가로, functional correctness와 reference 기반 similarity가 있다.

### Functional Correctness
> 의도한 기능을 수행하는지로 평가하며, 코드 생성·게임 봇 등에서 자동화 가능하다.

코드 생성은 test case 실행으로 검증한다(execution accuracy). HumanEval, MBPP, Spider(text-to-SQL) 등이 사용한다. 각 문제에 k개 샘플을 생성해 하나라도 모든 test case를 통과하면 해결로 보는 pass@k 지표를 쓴다(k가 클수록 점수↑).

### Similarity Measurements Against Reference Data
> reference response와 비교해 유사도를 측정하며, 4가지 방식이 있다.

(input, reference responses) 형식의 데이터가 필요해 reference 생성에 병목이 있다. 4방식: evaluator 판단, exact match(짧은 정답에 적합), lexical similarity(BLEU/ROUGE, edit distance/n-gram 중첩), semantic similarity(embedding 기반, cosine similarity). lexical은 포괄적 reference가 필요하고 높은 점수가 항상 좋은 응답은 아니다. semantic은 embedding 품질에 의존한다.

### Introduction to Embedding
> embedding은 원본 데이터의 의미를 담은 수치 벡터(보통 100~10,000차원)다.

유사한 데이터일수록 embedding이 가깝다. BERT, CLIP, Sentence Transformers 등이 생성한다. 다른 modality를 한 공간에 매핑하는 joint/multimodal embedding(CLIP, ULIP, ImageBind)이 새 영역이며, text 기반 이미지 검색 등을 가능케 한다. MTEB(Massive Text Embedding Benchmark)로 품질을 평가한다.

---

## 4. AI as a Judge
> AI로 AI를 평가하는 방식으로, production에서 가장 흔한 평가법 중 하나가 되었다.

### Why & How
> 빠르고 저렴하며 reference 없이 어떤 기준으로도 평가 가능하고, 판단 이유까지 설명한다.

GPT-4와 인간의 일치율(85%)이 인간 간 일치율(81%)보다 높다는 연구도 있다. 사용법: 응답 자체 평가, reference와 비교, 두 응답 비교. judge prompt는 task, 기준, 채점 체계(classification/discrete/continuous)를 명확히 해야 한다. 텍스트>숫자, classification>numerical, discrete>continuous가 더 잘 작동하며 예시 포함이 유리하다. AI judge는 model+prompt로 이뤄진 시스템이다.

### Limitations
> 확률적이라 inconsistent하고, 기준이 비표준이며, 비용·latency·편향이 있다.

inconsistency(같은 입력에 다른 점수), criteria ambiguity(도구마다 faithfulness 정의·점수 체계가 달라 비교 불가), 비용·latency 증가가 문제다. 편향: self-bias(자기 응답 선호), first-position bias, verbosity bias(긴 응답 선호). "model과 prompt를 볼 수 없는 AI judge는 신뢰하지 말 것."

### What Models Can Act as Judges?
> judge는 평가 대상보다 강하거나, 약하거나, 같을 수 있다.

강한 judge는 더 나은 판단을 주지만 비용·latency 때문에 일부만 평가에 쓴다. self-evaluation은 sanity check·자기 수정에 유용하다. 작은 특화 judge가 유망하며, reward model(점수), reference-based judge(BLEURT, Prometheus), preference model(PandaLM, JudgeLM; 어느 응답을 선호할지 예측)이 예다.

---

## 5. Ranking Models with Comparative Evaluation
> 모델을 개별 평가(pointwise)하거나 서로 비교(comparative)해 순위를 매긴다.

주관적 품질에는 비교가 더 쉽다. 각 비교(match)에서 evaluator가 승자를 고르고, win rate를 모아 Elo, Bradley-Terry, TrueSkill 같은 rating 알고리즘으로 순위를 계산한다(LMSYS Chatbot Arena). 순위는 미래 match 결과를 예측하는 문제로, 정답 순위는 없다. 단, 정답이 필요한 질문(예: 의학적 사실)을 preference로 답해선 안 된다.

```
matches -> win rates -> rating algorithm (Elo/Bradley-Terry) -> ranking
```

### Challenges & Future
> scalability, 표준화·품질관리 부재, 절대 성능 미파악이 한계다.

비교 수가 모델 수의 제곱으로 증가하나 transitivity 가정으로 완화한다(AI에 성립하는지는 불확실). 크라우드소싱은 다양하지만 표준화·품질관리가 어렵다(단순 prompt 다수, 사실 검증 부족). comparative는 어느 모델이 나은지는 알려주나 얼마나 좋은지·충분한지는 모른다. 그럼에도 포화되지 않고 gaming이 어려우며 human preference를 포착해 미래가 밝다.

---

## Summary (핵심 정리)
- foundation model은 open-ended·black-box·빠른 benchmark 포화로 평가가 어려우며, 평가 투자가 개발에 비해 뒤처져 있다.
- 평가법은 exact(functional correctness, similarity)와 subjective(AI as a judge)로 나뉘며, AI judge는 judge에 의존적이라 exact·human 평가로 보완해야 한다.
- 개별 평가 대신 comparative evaluation으로 순위를 매길 수 있으며, preference 신호 수요가 preference model 개발을 촉진한다.
