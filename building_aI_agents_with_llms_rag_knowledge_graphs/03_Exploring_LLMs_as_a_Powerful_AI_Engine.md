# 03. Exploring LLMs as a Powerful AI Engine

## 챕터 개요 (3줄 요약)

- 트랜스포머를 대규모로 학습하면 LLM(Large Language Model)이 되며, 스케일링 법칙과 창발적 속성을 다룬다.
- LoRA, 어댑터, RLHF, 명령어 튜닝 등 효율적 파인튜닝·정렬 기법과 경량화(양자화, 가지치기)를 설명한다.
- 멀티모달 모델(ViT, CLIP, BLIP-2, Stable Diffusion), 환각·윤리 문제, 프롬프트 엔지니어링까지 살펴본다.

---

## 1. Discovering the evolution of LLMs

> LLM은 100억 개 이상의 파라미터를 가진 트랜스포머로, 규모가 커지면 새로운 능력이 창발한다.

- LLM은 보통 100억 파라미터 이상이며, 대부분 다음 단어 예측(autoregressive language modeling)으로 학습한다.
- 파라미터 증가 동기: 학습성(learnability), 표현력(expressiveness), 기억력(memory).
- 스케일링 법칙(scaling law)은 모델 크기(N), 데이터 크기(D), 연산량(C)에 따라 손실이 매끄럽게 감소함을 보인다.
- DeepMind의 Chinchilla는 토큰 수가 더 중요하다고 주장하며, 현재 LLM들은 과소적합(underfitted) 상태일 수 있다.
- 합성 데이터로 학습 시 모델 붕괴(model collapse)나 파국적 망각(catastrophic forgetting) 위험이 있다.
- 창발적 속성(emergent properties)은 임계 규모(critical scale)에서 갑자기 나타나는 상전이(phase transition) 현상이다.
- 컨텍스트 길이(context length)가 길수록 장거리 의존성을 포착하나 연산 비용이 제곱으로 증가한다.
- MoE(Mixture of Experts)는 희소 연산(sparse computation)으로 같은 연산 예산에 더 큰 모델을 학습한다.

### MoE 라우팅 구조

```
token -> [Router / Gate] -> select top-k experts
   expert_1 (FFN)  expert_2 (FFN) ... expert_8 (FFN)
   only selected experts active (sparse)
```

---

## 2. Instruction tuning, fine-tuning, and alignment

> 100억+ 파라미터 모델의 파인튜닝 비용을 줄이기 위해 가중치를 동결하고 일부만 학습하는 효율적 기법을 사용한다.

- LoRA(Low-Rank Adaptation)는 가중치 변화 ∆W를 저차원 행렬 A·B로 근사해 적은 파라미터만 학습한다.
- LoRA는 추론 시 비용 증가가 없고 원래 능력을 손상하지 않으며 도메인별 변화 행렬을 따로 만들 수 있다.
- 어댑터(adapter)는 트랜스포머 블록 안에 오토인코더 구조의 작은 층을 추가해 약 3.6% 파라미터만 학습한다.
- 프롬프트 튜닝, 프리픽스 튜닝도 있으나 불안정해 LoRA와 어댑터가 가장 널리 쓰인다.
- 정렬(alignment)은 모델을 인간 가치(helpful, honest, harmless)에 맞추는 추가 학습이다.
- RLHF(Reinforcement Learning from Human Feedback)는 SFT, 보상 모델 학습, PPO(Proximal Policy Optimization) 세 단계로 구성된다.
- DPO(Direct Preference Optimization)는 보상 모델 없이 더 나은/나쁜 완성을 비교해 강화학습을 회피한다.
- 명령어 튜닝(instruction tuning)은 명령-출력 쌍으로 학습해 미학습 작업에도 적응하는 능력을 키운다.

---

## 3. Exploring smaller and more efficient LLMs

> 많은 실무 사례에서는 100억 파라미터 모델이 불필요하며, 작고 효율적인 SLM으로 충분하다.

- SLM(Small Language Model)은 자원을 적게 쓰고 상용 GPU나 CPU, 휴대폰에서도 동작 가능하다.
- 얕은 모델은 문법은 좋으나 일관성이 떨어지며, 일관성·창의성에는 더 많은 층과 은닉 크기가 필요하다.
- 소형 LLM 확보 방법: 처음부터 학습(Mistral 7B 등), 지식 증류, 기존 모델 크기 축소.
- 양자화(quantization)는 가중치를 고정밀(float)에서 저정밀(int) 자료형으로 매핑해 메모리를 줄인다.
- 아핀 양자화(affine quantization)는 스케일·제로 포인트 두 인자로 정밀도를 낮추고 클리핑을 수행한다.
- 가지치기(pruning)는 불필요한 가중치·연결·층을 제거하며 비정형(unstructured)과 정형(structured)으로 나뉜다.
- SparseGPT는 재학습 없이 1700억 파라미터 모델을 최대 60%까지 압축한다.
- 큰 모델일수록 중복 층이 많아 깊은 층을 제거해도 성능 저하가 적다.

---

## 4. Exploring multimodal models

> 데이터를 벡터로 변환할 수 있으면 트랜스포머에 입력할 수 있어 이미지·음악 등 다양한 모달리티로 확장 가능하다.

- ViT(Vision Transformer)는 이미지를 16x16 패치로 나눠 토큰처럼 처리하는 인코더 전용 모델이다.
- CLIP(Contrastive Language-Image Pre-Training)는 이미지·텍스트를 같은 공간에 임베딩하는 대조 학습(contrastive learning) 모델이다.
- CLIP은 4억 개 (이미지, 텍스트) 쌍으로 학습하며 코사인 유사도를 최대화/최소화한다.
- CLIP은 제로샷 분류(zero-shot classification), 검색, 클러스터링에 쓰이나 텍스트 생성은 못 한다.
- VLM(Vision-Language Model)인 BLIP-2(Bootstrapping Language-Image Pre-training)는 LLM과 ViT를 Q-Former로 연결한다.
- Stable Diffusion은 텍스트 인코더, U-Net 이미지 생성기, 이미지 디코더로 구성된 text-to-image 모델이다.
- 확산 과정(diffusion)은 노이즈를 추가/예측하며 학습하고, U-Net의 크로스 어텐션으로 텍스트 조건을 반영한다.

### Stable Diffusion 흐름

```
text -> [Text Encoder (CLIP)] -> text embedding
noise -> [U-Net diffusion + cross-attention] -> latent
latent -> [Image Decoder] -> image
```

---

## 5. Understanding hallucinations and ethical and legal issues

> LLM은 사실과 어긋나거나 지시에 맞지 않는 환각(hallucination)을 생성할 수 있고 편향·윤리·법적 위험을 동반한다.

- 환각은 사실성 환각(factuality)과 충실성 환각(faithfulness)으로 나뉜다.
- 모델은 독성 콘텐츠와 특정 집단에 대한 고정관념을 생성할 수 있다(representational harm).
- 자원을 불공정하게 배분하는 할당적 피해(allocational harm)도 발생할 수 있다.
- 편향은 사전 학습 데이터에서 비롯되므로 학습 전 문제 콘텐츠 제거(detoxify)가 중요하다.
- LLM은 허위정보·피싱 이메일 등 악용 위험이 있어 생성 텍스트 탐지·워터마킹이 연구된다.
- 저작권 문제(fair use 논쟁)와 학습 데이터 유출 같은 프라이버시 위험이 존재한다.
- 머신 언러닝(machine unlearning)으로 개인정보를 잊게 하는 방법이 연구되고 있다.

---

## 6. Prompt engineering

> ICL(In-Context Learning)은 파라미터 갱신 없이 프롬프트의 예시만으로 새 작업을 수행하는 LLM의 창발적 속성이다.

- ICL은 "Language Models are Few-Shot Learners" 논문에서 정의되었으며 진짜 학습이 아닌 잠재 표현 활용이다.
- 제로샷(zero-shot) 프롬프트는 예시 없이 질문/지시만 제공한다.
- 퓨샷(few-shot) 프롬프트는 예시를 제공하며 3-shot, 5-shot 등이 흔하다.
- CoT(Chain-of-Thought)는 <입력, 사고 과정, 출력> 삼중항으로 추론 단계를 제공한다.
- "Let's think step by step"만 추가하는 제로샷 CoT도 효과적이다.
- 자기 일관성(self-consistency)은 여러 해답을 생성해 다수결로 선택하는 앙상블 기법이다.
- ToT(Tree of Thoughts)는 탐색 알고리즘으로 추론 중간 단계를 생성·평가한다.
- DSPy(Declarative Self-improving Language Programs in Python)는 프롬프트를 시그니처·모듈로 추상화해 자동 최적화한다.

---

## Summary (핵심 정리)

- 트랜스포머에서 LLM으로의 전환과 스케일링 법칙, 창발적 속성을 배웠다.
- self-attention은 강력한 표현 학습의 핵심이자 막대한 연산 비용이라는 한계의 원천임을 이해했다.
- 효율적 파인튜닝, 경량화, 멀티모달 확장, 프롬프트 기법으로 LLM을 실전 활용하는 법을 익혔다.
