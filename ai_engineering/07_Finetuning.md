# 07. Finetuning

## 챕터 개요 (3줄 요약)
- finetuning은 weight를 조정해 모델을 특정 작업에 적응시키는 transfer learning으로, 주로 instruction-following과 출력 형식을 개선한다.
- finetuning의 핵심 병목인 memory(파라미터·trainable 파라미터·numerical 표현)와 이를 줄이는 quantization을 다룬다.
- 메모리 효율적 기법인 PEFT(특히 LoRA), quantized LoRA, model merging, 실무 tactics를 설명한다.

---

## 1. Finetuning Overview
> 학습된 base model에서 출발해 특정 작업에 맞게 추가 학습하는 transfer learning이다.

transfer learning은 한 작업의 지식을 관련 작업으로 전이해 sample efficiency를 높인다. finetuning은 pre-training의 연장으로 여러 형태가 있다: self-supervised finetuning(continued pre-training), supervised finetuning(SFT, (input,output) 쌍), preference finetuning(comparative data), long-context finetuning(positional embedding 수정), infilling finetuning(빈칸 채우기).

---

## 2. When to Finetune
> prompt-based 방법을 충분히 시도한 후 finetuning을 고려한다.

### Reasons to Finetune
> 주로 품질 향상, 구조화 출력, 도메인 특화, bias 완화에 쓴다.

general-purpose 모델이 특정 작업에 약할 때 유용하다. bias 완화(여성 CEO 데이터로 finetune), distillation(큰 모델 모방으로 작은 모델 학습)이 흥미로운 사례다. 작은 모델 finetuning이 더 흔하다(메모리·비용·속도).

### Reasons Not to Finetune
> 한 작업 개선이 다른 작업 성능을 저하시킬 수 있고, 높은 선투자·유지보수가 필요하다.

데이터 확보, 학습 지식, 서빙, 모니터링·업데이트 정책이 필요하다. 새 base model이 빠르게 나와 finetuned 모델을 능가할 수 있다. 실험은 prompting부터 체계적으로 시작해야 한다. general 모델이 도메인 모델을 능가하기도 한다(GPT-4 > BloombergGPT).

### Finetuning and RAG
> 실패가 정보 기반이면 RAG, 행동 기반이면 finetuning. "finetuning은 form, RAG는 facts."

정보 부족(사실 오류·outdated)은 RAG가 우수하다. 행동 문제(관련성 부족, 형식 미준수, semantic parsing)는 finetuning이 돕는다. 둘 다 문제면 RAG부터 시작(BM25 같은 단순 검색). RAG와 finetuning은 함께 쓸 수 있다. 워크플로: prompting → 예시 추가 → RAG(단순) → 고급 RAG 또는 finetuning → 결합.

---

## 3. Memory Bottlenecks
> foundation model 규모로 memory가 inference·finetuning의 병목이며, finetuning이 더 많이 필요하다.

### Backpropagation and Trainable Parameters
> trainable parameter 수가 finetuning memory를 결정한다.

backpropagation은 forward pass(출력 계산)와 backward pass(loss로 weight 갱신: gradient 계산, optimizer로 조정)로 구성된다. trainable parameter마다 gradient와 optimizer state 추가 메모리가 필요하다. frozen parameter는 갱신되지 않는다.

### Memory Math
> inference는 weight+activation, training은 추가로 gradient+optimizer state가 필요하다.

inference: N × M × 1.2 (활성화 ~20% 가정). 13B 모델(2 bytes/param) = 26GB × 1.2 = 31.2GB. training: weights + activations + gradients + optimizer states. Adam은 param당 2개 state. gradient checkpointing(activation 재계산)으로 메모리를 줄이나 시간이 늘어난다.

```
Inference memory = N * M * 1.2
Training memory  = weights + activations + gradients + optimizer states
Adam: trainable param 당 gradient 1 + optimizer state 2 = 3 values
```

### Numerical Representations & Quantization
> 값당 비트 수가 memory를 좌우하며, quantization(저정밀도 변환)이 효과적이다.

FP32(4 bytes), FP16/BF16(2 bytes), INT8/INT4 등. bit는 sign·range(exponent)·precision(significand)로 나뉜다. BF16은 FP16과 같은 비트지만 range가 넓고 precision이 낮다. quantization은 PTQ(Post-Training Quantization, 가장 흔함)와 training 중(QAT, Quantization-Aware Training)으로 나뉜다. inference는 8/4비트가 표준화되고, BitNet b1.58은 1.58비트로 16비트 Llama 2급 성능을 낸다. training은 정밀도에 민감해 mixed precision으로 한다.

---

## 4. Finetuning Techniques

### Parameter-Efficient Finetuning (PEFT)
> trainable parameter를 줄여 full finetuning에 근접한 성능을 낸다.

full finetuning(모든 param 갱신)은 7B 모델에 56GB+ 필요. partial finetuning(일부 층만)은 parameter-inefficient. PEFT는 적은 trainable param으로 full에 근접. 두 갈래: **adapter-based**(LoRA, BitFit, IA3 등 모듈 추가)와 **soft prompt-based**(학습 가능한 연속 벡터 토큰 추가; prefix-tuning, P-tuning, prompt tuning). LoRA가 압도적으로 인기.

### LoRA (Low-Rank Adaptation)
> weight 행렬을 두 작은 행렬의 곱으로 분해해 그것만 학습하며, inference latency를 추가하지 않는다.

W(n×m)를 A(n×r)·B(r×m)로 분해, W' = W + (α/r)·WAB. A, B만 갱신. low-rank factorization 기반. LLM은 intrinsic dimension이 낮아 적은 param·데이터로 finetune 가능하다. attention의 Wq·Wk·Wv·Wo에 적용(query·value가 효과적), r은 4~64면 충분. 서빙: 병합(단일 모델, latency 무) vs 분리(multi-LoRA 서빙, 저장 절약).

```
W' = W + (alpha/r) * (A x B)    # A: n x r, B: r x m, 학습은 A,B만
```

### Quantized LoRA & Model Merging
> QLoRA는 weight를 4비트로 저장해 메모리를 더 줄인다.

QLoRA는 NF4(NormalFloat-4)로 저장, 계산 시 BF16으로 dequantize, paged optimizer 사용 → 65B를 단일 48GB GPU에서 finetune. **Model merging**은 여러 모델을 결합해 더 나은 모델을 만든다(multi-task finetuning, on-device, federated learning). ensemble(출력 결합)과 달리 parameter를 결합한다. 세 방식: **summing**(linear combination, SLERP; task vector로 task arithmetic; TIES/DARE로 redundant param pruning), **layer stacking**(frankenmerging, MoE 생성, depthwise scaling 업스케일), **concatenation**(rank 합산, 비권장).

---

## 5. Finetuning Tactics
> base model, finetuning method, framework를 선택한다.

가장 강한 모델로 feasibility를 확인 후 약한 모델 탐색. OpenAI의 progression path(저렴한 모델로 코드 테스트→중간 모델로 데이터 테스트→최고 모델 실험)와 distillation path(강한 모델로 소량 학습→데이터 생성→저렴한 모델 학습). LoRA부터 시작, 데이터가 적으면 PEFT가 유리. framework: finetuning API(간단하나 제약), LLaMA-Factory·unsloth·Axolotl 등(유연), 분산학습은 DeepSpeed 등.

### Hyperparameters
> learning rate, batch size, epochs, prompt loss weight를 조정한다.

**learning rate**(보통 1e-7~1e-3; loss 곡선이 요동치면 너무 큼, 느리면 작음; learning rate schedule). **batch size**(작으면 불안정; 메모리 제약; gradient accumulation으로 보완). **epochs**(작은 데이터는 더 많이; train loss↓ val loss↑면 overfitting). **prompt loss weight**(prompt가 loss에 기여하는 비율, 보통 10%).

---

## Summary (핵심 정리)
- finetuning은 transfer learning으로 form(형식·스타일)을 개선하며, facts가 필요하면 RAG를 쓴다("finetuning은 form, RAG는 facts").
- full finetuning은 메모리가 커 비현실적이라, PEFT(LoRA, trainable param 감소)와 quantization(비트 감소)으로 메모리 병목을 완화한다.
- LoRA는 parameter·sample 효율적이고 모듈식이라 multi-LoRA 서빙·결합이 쉬우며, model merging으로 여러 모델을 하나로 합칠 수 있다.
