# 01. Introduction to Building AI Applications with Foundation Models

## 챕터 개요 (3줄 요약)
- 2020년 이후 AI의 핵심은 'scale'이며, self-supervision으로 거대화된 language model이 multimodal foundation model로 확장되면서 AI engineering이 탄생했다.
- foundation model이 가능케 한 다양한 use case(coding, 이미지/영상, writing, 교육, chatbot, 정보 집약, 데이터 정리, workflow 자동화)를 살펴본다.
- AI 애플리케이션을 만들기 전 고려사항(왜 만드는가, 방어력, 기대치 설정)과, ML engineering과 구분되는 AI engineering stack을 설명한다.

---

## 1. The Rise of AI Engineering

> language model에서 LLM(Large Language Model), 그리고 foundation model을 거쳐 AI engineering이 등장한 흐름.

### From Language Models to Large Language Models
> language model은 언어의 통계 정보를 인코딩하며, self-supervision 덕분에 LLM 규모로 확장될 수 있었다.

기본 단위는 token이며 tokenization으로 텍스트를 분해한다. masked LM(BERT처럼 앞뒤 문맥으로 빈칸 채우기, 비생성 작업)과 autoregressive LM(앞 token만으로 다음 token 예측, 생성의 표준) 두 종류가 있다. 핵심은 self-supervision: 입력 데이터 자체에서 label(다음 token)을 추론해 labeling 비용 없이 대규모 학습이 가능하다. 모델 크기는 parameter 수로 측정하며, 큰 모델일수록 더 많은 학습 데이터가 필요하다.

```
Autoregressive LM:  [t1 t2 t3] -> predict t4   (uses only preceding)
Masked LM:          [t1 __ t3] -> predict t2    (uses both sides)
```

### From Large Language Models to Foundation Models
> 텍스트를 넘어 여러 modality를 처리하는 multimodal model로 확장되며 task-specific에서 general-purpose 모델로 전환됐다.

NLP(Natural Language Processing), computer vision 등 modality별로 나뉘던 AI 연구가 foundation model로 통합됐다. multimodal model(LMM, Large Multimodal Model)은 text+image 등 여러 token에 조건부로 생성한다. CLIP은 natural language supervision으로 4억 (image,text) 쌍을 학습한 embedding model이다. general-purpose 모델은 prompt engineering, RAG(Retrieval-Augmented Generation), finetuning으로 특정 작업에 맞춰진다.

### From Foundation Models to AI Engineering
> AI engineering은 기존 foundation model 위에 애플리케이션을 구축하는 과정이다.

세 요인이 급성장을 이끈다: (1) general-purpose AI 능력으로 가능 작업·수요 폭증, (2) ChatGPT 이후 AI 투자 급증, (3) model-as-a-service(API)로 진입 장벽 하락. 전통 ML engineering이 모델 개발이라면, AI engineering은 기존 모델 활용·적응에 집중한다.

---

## 2. Foundation Model Use Cases
> consumer/enterprise 전반에서 거의 모든 작업에 적용 가능하며, 8개 범주로 분류된다.

가장 인기 있는 use case는 coding이다. 그 외 image/video production(Midjourney 등 창작), writing(완성 학습 덕분에 강점, SEO·이메일·마케팅), education(개인화 학습·튜터·퀴즈), conversational bots(고객지원·동반자·3D NPC), information aggregation(요약·talk-to-your-docs), data organization(비정형 데이터에서 구조화 정보 추출, IDP), workflow automation(반복 작업·agent)이 있다. enterprise는 위험이 낮은 내부용 애플리케이션을 먼저 배포하는 경향이 있다.

---

## 3. Planning AI Applications
> 멋진 데모는 쉽지만 수익성 있는 제품은 어렵다. 만들기 전에 왜·어떻게를 따져야 한다.

### Use Case Evaluation
> 위험과 기회의 관점에서 왜 만드는지 평가한다.

위험 수준은 (높음) 미적용 시 경쟁사에 도태(business continuity), 이익·생산성 기회 상실, 뒤처지지 않기 위한 탐색(낮음) 순이다. 존재적 위협이면 in-house, 생산성 향상이면 buy 옵션도 많다. AI의 역할은 critical/complementary, reactive/proactive, dynamic/static으로 구분되며, human-in-the-loop 정도(Crawl-Walk-Run)도 결정해야 한다.

### AI Product Defensibility
> 진입 장벽이 낮아 방어력(moat)이 중요하다.

기반 모델 능력이 확장되면 당신의 layer가 흡수될 수 있다. 경쟁 우위는 technology, data, distribution 셋이며, foundation model 시대엔 기술이 비슷해 data가 핵심 moat가 될 수 있다(빠른 출시로 사용 데이터 축적).

### Setting Expectations & Milestone Planning & Maintenance
> 성공 지표와 usefulness threshold를 정의하고, last mile 문제와 빠른 변화에 대비한다.

비즈니스 지표(자동화 비율, 처리량, 응답 속도, 인건비 절감)와 usefulness threshold(quality, latency: TTFT/TPOT/total, cost, 공정성)를 설정한다. "60까지는 쉽고 100은 매우 어렵다"는 last mile 문제가 있다. 유지보수는 AI의 빠른 변화(가격 하락, API 수렴, 규제·IP)에 대응해야 한다.

---

## 4. The AI Engineering Stack
> AI engineering은 ML engineering에서 진화했으며, 모델 개발보다 적응·평가에 집중한다.

### Three Layers of the AI Stack
> application development, model development, infrastructure 세 계층으로 구성된다.

application development는 prompt·context 제공과 평가·인터페이스를 다루며 최근 가장 활발하다. model development는 modeling/training, dataset engineering, inference optimization을 포함한다. infrastructure는 serving, 데이터·compute 관리, monitoring을 담당하며 변화가 가장 적다.

```
+------------------------------+
| Application Development       |  prompt, evaluation, AI interface
+------------------------------+
| Model Development             |  training, dataset eng., inference opt.
+------------------------------+
| Infrastructure                |  serving, compute/data, monitoring
+------------------------------+
```

### AI Engineering vs ML Engineering
> 세 가지 큰 차이: 모델 학습 대신 적응, 더 큰 compute·inference 최적화 압박, open-ended 출력으로 평가가 어려움.

모델 적응은 weight 변경 없는 prompt-based 기법과 weight를 갱신하는 finetuning으로 나뉜다. model development는 modeling/training(ML 지식은 nice-to-have), dataset engineering(feature engineering보다 dedup·tokenization·retrieval·품질관리), inference optimization(더 중요해짐)으로 구성된다. training 용어는 pre-training(처음부터), finetuning(이어서 학습), post-training(개발자가 출시 전 수행)으로 구분된다.

### Application Development & Full-Stack
> 같은 모델을 쓰므로 차별화는 application development에서 나온다.

evaluation(open-ended 특성으로 더 어렵고 중요), prompt engineering/context construction(weight 변경 없이 원하는 동작 유도), AI interface(standalone 앱·브라우저 확장·chat 통합·plug-in)가 핵심이다. 인터페이스 중요성이 커지며 AI engineering이 full-stack에 가까워지고, "제품 먼저 만들고 가능성이 보이면 데이터·모델에 투자"하는 빠른 반복이 보상받는다.

---

## Summary (핵심 정리)
- self-supervision으로 LM이 LLM이 되고, 다른 modality를 통합해 foundation model이 되었으며, 이것이 AI engineering을 탄생시켰다.
- foundation model은 consumer/enterprise 전반의 수많은 use case를 가능케 했지만, "만들어야 하는가"를 먼저 따지고 방어력·기대치·유지보수를 계획해야 한다.
- AI engineering은 ML engineering의 많은 원칙을 계승하되, 모델 개발보다 적응·평가에 집중하며 점점 product와 full-stack에 가까워진다.
