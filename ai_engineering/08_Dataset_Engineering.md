# 08. Dataset Engineering

## 챕터 개요 (3줄 요약)
- 모델 품질은 데이터 품질에 달려 있으며, dataset engineering은 예산 내 최고 모델을 위한 데이터셋을 만드는 것이다.
- data curation의 세 기준(quality, coverage, quantity)과 데이터 획득·annotation을 다룬다.
- data augmentation/synthesis(특히 AI 기반), model distillation, data processing(검사·중복제거·정제·포맷)을 설명한다.

---

## 1. Data Curation
> 모델이 어떻게 학습하는지와 가용 자원을 이해해야 하는 과학으로, quality·coverage·quantity 세 기준을 따른다.

작업에 따라 데이터 형식이 다르다: self-supervised(시퀀스), instruction(instruction,response), preference(instruction,winning,losing). CoT(Chain-of-Thought) reasoning, tool use 같은 복잡한 행동은 데이터 확보가 더 어렵다(인간과 AI의 작업 방식이 달라 simulation·synthetic이 필요). single-turn vs multi-turn도 고려한다. 나쁜 행동을 잊게 하려 데이터를 제거하기도 한다.

### Data Quality
> 소량의 고품질 데이터가 대량의 noisy 데이터를 능가한다.

Yi, LIMA(1,000개 큐레이션 prompt), Llama 3가 이를 입증했다. 고품질의 6가지 특성: relevant(작업 관련), aligned(작업 요구 부합), consistent(annotator 간 일관), correctly formatted(모델 기대 형식), sufficiently unique(중복 최소), compliant(정책·법규 준수).

### Data Coverage
> 학습 데이터는 예상 문제 범위를 포괄해야 하며, 충분한 diversity가 필요하다.

사용 패턴의 다양성(상세/짧은 instruction, 오타, 프로그래밍 언어 등)을 담아야 한다. 애플리케이션마다 diversity 축이 다르다. Llama 3는 학습 단계별로 최적 domain mix가 다르며(pre-training은 math·code가 ~50%), high-quality·diverse 데이터가 둘 중 하나만인 데이터를 능가한다.

### Data Quantity
> 필요량은 상황마다 크게 다르며, finetuning 기법·작업 복잡도·base model 성능에 좌우된다.

full finetuning은 PEFT보다 훨씬 많은 데이터 필요(수만~수백만 vs 수백~수천). 강한 모델일수록 적은 예시로 finetune 가능(데이터 많으면 모델 간 차이 줄어듦). 소량(50개)으로 시작해 개선 여부를 보고, performance gain curve로 추가 데이터 효과를 추정한다. task 다양성도 중요하다(9→282 task에서 성능 급증). ossification 주의.

---

## 2. Data Acquisition and Annotation
> 예산 내에서 충분히 크고 품질·다양성을 갖춘 데이터를 프라이버시·규정 준수하며 확보한다.

가장 중요한 출처는 자체 애플리케이션 데이터(data flywheel)다. 공개 데이터셋(Hugging Face, Kaggle, Data.gov 등)을 먼저 확인하되 license를 검증하고 철저히 검사한다. 여러 출처를 mix-and-match한다. annotation은 과정 자체보다 명확한 annotation guideline 작성이 가장 어렵다(평가 데이터 guideline과 동일).

---

## 3. Data Augmentation and Synthesis
> augmentation은 실제 데이터에서 새 데이터를 파생, synthesis는 실제 데이터 속성을 모방해 생성한다.

### Why Data Synthesis
> quantity·coverage·quality 향상, 프라이버시 완화, model distillation을 위해 쓴다.

대규모 데이터 생성, 타겟 특성 데이터(adversarial, rare class), 때로 인간보다 높은 품질(tool use, 복잡한 수학, 일관된 preference), 민감 데이터 대체(의료·보험), 큰 모델 모방(distillation)이 동기다.

### Traditional & AI-Powered Synthesis
> rule-based(템플릿), simulation, 그리고 AI 기반 생성.

**Rule-based**: 템플릿+random generator(Faker), 이미지 변환(회전·크롭), 단어 치환(bias 완화), perturbation(노이즈로 robustness↑). **Simulation**: 자율주행·로봇·rare event(금융·기후), Sim2Real. **AI-powered**: API 시뮬레이션, self-play(Dota 2, AlphaGo), paraphrasing/translation(back-translation 검증), 저자원 언어, 코드 번역.

### Instruction Data Synthesis & Verification
> AI로 instruction·response·둘 다 생성한다.

토픽 리스트나 템플릿에서 instruction 생성(UltraChat, Alpaca). reverse instruction(고품질 콘텐츠로 prompt 역생성)으로 hallucination 회피. Llama 3는 code translation·back-translation·생성+검증(parser·linter·unit test·오류수정) 파이프라인으로 270만 코딩 예시 생성. **검증**: functional correctness, AI judge, 휴리스틱 필터링.

### Limitations
> 품질 관리, 피상적 모방, model collapse, 데이터 lineage 불명확.

저품질("garbage in, garbage out"), superficial imitation(스타일만 모방, hallucination 강요), model collapse(재귀 학습으로 성능 저하; 실제 데이터와 혼합으로 완화), bias 증폭, 데이터 lineage 가림(저작권·벤치마크 오염 위험)이 한계다.

### Model Distillation
> 작은 student 모델이 큰 teacher 모델을 모방하도록 학습한다.

배포용 소형 모델 생성이 목적(DistilBERT: 40% 작고 97% 성능 유지). Alpaca는 text-davinci-003 출력으로 Llama-7B를 finetune. 단, synthetic 학습이 모두 distillation은 아니며(Nemotron-4는 student가 teacher 능가), license가 출력 사용을 금지하기도 한다.

---

## 4. Data Processing
> 각 use case 요구에 맞게 데이터를 처리한다.

### Inspect Data
> 데이터 품질 감을 잡기 위해 먼저 검사한다.

출처·통계·분포(토큰, 길이, 토픽, 언어)를 파악하고, 출처·시간·annotator별로 분포를 본다. inter-annotator disagreement를 계산한다. "수동 데이터 검사는 ML에서 가장 가성비 높은 활동"—15분만 봐도 통찰을 얻는다.

### Deduplicate Data
> 중복은 분포를 왜곡하고 bias·test 오염을 일으킨다.

whole/intra/cross-document 중복 등 여러 형태가 있다. 기법: pairwise comparison, hashing(MinHash, Bloom filter), dimensionality reduction. dupeGuru, Dedupe 등 라이브러리 활용.

### Clean, Filter & Format Data
> 성능과 안전을 위해 정제·필터·포맷한다.

extraneous 포맷 토큰(HTML/Markdown) 제거, 비준수 데이터(PII·저작권·toxic) 제거, 저품질 데이터 제거. active learning·importance sampling·data pruning으로 추가 필터링. 모델별 tokenizer·chat template에 맞게 포맷한다(잘못된 template은 버그 유발). finetuning 후엔 짧은 prompt 사용 가능(비용 절감).

---

## Summary (핵심 정리)
- 데이터셋 설계는 모델이 학습할 행동을 정의하고 그것을 보여주는 데이터를 만드는 것으로, 학습 단계와 무관하게 quality·coverage·quantity 세 기준을 따른다.
- 소량의 고품질·다양한 데이터가 대량의 noisy 데이터를 능가하며, 고품질 데이터 확보의 어려움으로 synthetic 데이터가 실용화됐다.
- synthetic 데이터도 평가가 필요하며, annotation guideline 작성·검증 등 자동화가 어려운 창의적 작업이 데이터 엔지니어링의 핵심이다.
