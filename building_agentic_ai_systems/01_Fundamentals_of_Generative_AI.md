# 01. Fundamentals of Generative AI

## 챕터 개요 (3줄 요약)

- 생성형 AI(Generative AI)의 정의와 간략한 역사를 다루며, 콘텐츠를 새롭게 만들어내는 기술이 산업 전반과 에이전트 기반 지능형 시스템의 자율성을 확장하고 있음을 설명한다.
- VAE, GAN, 자기회귀(Autoregressive), Transformer 등 주요 생성 모델의 작동 원리와 차이를 정리한다.
- 의료·금융·교육·미디어 등 응용 분야와 데이터 편향·프라이버시·컴퓨팅 자원·윤리 같은 한계 및 도전 과제를 짚는다.

---

## 1. Introduction to generative AI

> 생성형 AI는 학습 데이터와 입력(주로 텍스트 프롬프트)을 바탕으로 텍스트·이미지·오디오·비디오 등 새로운 콘텐츠를 만들어내는 AI 기술 부류다.

- 생성형 AI는 기존 데이터를 분류·예측하는 데 그치지 않고, 학습한 분포를 바탕으로 **새로운 샘플**을 생성한다는 점에서 전통적 판별형(discriminative) 모델과 구분된다.
- 초기에는 규칙 기반·통계 기반 접근에서 출발했으나, 딥러닝(Deep Learning)의 발전으로 표현력과 품질이 비약적으로 향상되었다.
- 입력 파라미터에는 텍스트 프롬프트가 일반적으로 포함되며, 멀티모달(multimodal) 입력도 점차 확대되고 있다.
- LLM(Large Language Model)의 등장으로 자연어 기반 상호작용과 추론 능력이 크게 강화되었다.
- 이 기술은 창의적 작업과 문제 해결을 자동화하며, 에이전트 기반 지능형 시스템(agentic system)의 핵심 구성 요소로 자리잡고 있다.
- AI(Artificial Intelligence) 및 ML(Machine Learning) 분야에서 가장 빠르게 성장하는 영역 중 하나다.

---

## 2. Types of generative AI models

> 대표적인 생성 모델은 VAE, GAN, 자기회귀 모델, Transformer 기반 모델이며 각기 다른 방식으로 데이터 분포를 학습한다.

- 생성 모델은 데이터의 잠재 구조(latent structure)를 학습하여 그로부터 새로운 데이터를 샘플링한다.
- 모델 유형에 따라 학습 안정성, 생성 품질, 다양성(diversity) 사이의 트레이드오프가 존재한다.
- 최근에는 Transformer 기반 LLM이 텍스트뿐 아니라 멀티모달 생성의 중심 아키텍처로 부상했다.

### VAE (Variational Autoencoder)

- VAE는 입력을 잠재 공간(latent space)으로 인코딩한 뒤 다시 디코딩하여 재구성하는 구조다.
- 잠재 변수를 확률 분포로 모델링해 연속적이고 부드러운 잠재 공간을 학습한다.
- 생성 품질은 GAN보다 다소 흐릿할 수 있으나 학습이 안정적이고 잠재 표현 해석이 용이하다.

### GAN (Generative Adversarial Network)

- 생성자(Generator)와 판별자(Discriminator)가 적대적으로 경쟁하며 학습하는 구조다.
- 생성자는 진짜 같은 데이터를, 판별자는 진짜/가짜를 구분하도록 훈련된다.
- 고품질 이미지 생성에 강력하지만 학습 불안정성(mode collapse 등) 문제가 있다.

```
[Random Noise z] -> (Generator) -> [Fake Sample]
                                        |
[Real Sample] ----------------------> (Discriminator) -> Real / Fake
        ^----------- adversarial feedback -----------|
```

### Autoregressive models and Transformer architecture

- 자기회귀 모델은 이전 토큰을 조건으로 다음 토큰을 순차적으로 예측한다.
- Transformer는 self-attention 메커니즘으로 장거리 의존성(long-range dependency)을 효율적으로 포착한다.
- 현재 대부분의 LLM(Large Language Model)은 Transformer 디코더 기반으로 구축된다.

```
Input Tokens -> [Embedding] -> [Self-Attention] -> [Feed-Forward] -> Next Token
                                   (repeated N layers)
```

### LLM-powered AI agents

- LLM은 추론·계획·도구 사용을 수행하는 에이전트(agent)의 두뇌 역할을 한다.
- 자연어 이해와 생성 능력을 바탕으로 복잡한 작업을 분해하고 실행한다.
- 이는 이후 챕터에서 다룰 agentic system의 기반이 된다.

---

## 3. Applications of generative AI

> 생성형 AI는 의료·금융·교육·미디어·마케팅·제조·리테일 등 다양한 산업에서 콘텐츠 생성과 자동화를 가능하게 한다.

- **이미지/비디오 생성**: 멀티모달 모델로 시각효과, 아바타, VR(Virtual Reality) 콘텐츠, 패션 디자인 등을 생성한다.
- **텍스트 생성**: 카피라이팅, 요약, 번역, 챗봇, 코드 생성 등 광범위한 언어 작업을 수행한다.
- **의료**: 신약 후보 물질 탐색, 의료 영상 보강, 합성 데이터 생성에 활용된다.
- **금융**: 리포트 자동 생성, 시나리오 시뮬레이션, 고객 응대 자동화에 적용된다.
- **교육**: 맞춤형 학습 자료 생성과 개인화 튜터링을 지원한다.
- **마케팅**: 광고·프로모션 콘텐츠를 대규모로 빠르게 제작한다.

---

## 4. Challenges and limitations of generative AI

> 생성형 AI는 데이터 품질·편향, 프라이버시, 컴퓨팅 자원, 윤리, 일반화 한계 등 여러 도전 과제를 안고 있다.

- **데이터 품질과 편향(bias)**: 학습 데이터의 편향이 출력에 그대로 반영되어 불공정하거나 왜곡된 결과를 낳을 수 있다.
- **데이터 프라이버시**: 학습 데이터에 포함된 민감 정보가 의도치 않게 노출될 위험이 있다.
- **컴퓨팅 자원**: 대규모 모델의 학습과 추론에는 막대한 연산·에너지 비용이 든다.
- **윤리적·사회적 영향**: 딥페이크, 허위정보, 저작권 문제 등 사회적 리스크가 존재한다.
- **일반화와 창의성**: 모델은 학습 분포를 벗어난 진정한 창의성보다는 패턴 재조합에 가까운 경향이 있다.

### 정리 포인트

- 한계는 기술적 측면(자원·일반화)과 사회적 측면(편향·윤리·프라이버시)으로 나눌 수 있다.
- 이러한 도전 과제는 신뢰할 수 있는(trustworthy) 에이전트 시스템 설계의 동기가 된다.

---

## Summary (핵심 정리)

- 생성형 AI는 학습된 분포로부터 새로운 콘텐츠를 만들어내는 기술로, VAE·GAN·Autoregressive·Transformer가 핵심 모델 군이다.
- 의료·금융·교육·미디어 등 폭넓은 산업에 적용되며, 특히 LLM은 자율적 에이전트 시스템의 핵심 엔진이다.
- 데이터 편향, 프라이버시, 컴퓨팅 비용, 윤리 같은 한계를 이해하는 것이 안전하고 신뢰할 수 있는 시스템 구축의 출발점이다.
