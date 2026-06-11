# 02. Large Language Model Serving

## 챕터 개요 (3줄 요약)
- LLM serving을 이해하려면 decoder-only Transformer의 autoregressive token 생성 과정과 attention 메커니즘을 serving 관점에서 파악해야 한다.
- prefill(prompt 일괄 처리, compute-intensive)과 decode(token 하나씩 생성, memory-intensive) 두 phase, 그리고 KV cache 재사용이 LLM serving 설계의 거의 모든 결정을 좌우한다.
- vLLM 같은 serving framework는 KV-cache 재사용·batching·streaming·concurrency를 제공해 HF Transformers 대비 10~20배 throughput을 달성한다.

---

## 1. Inside the Mind of a Transformer — LLM Evolution
> 언어모델은 rule-based → Word2Vec(2013, dense embedding) → RNN/LSTM/GRU → Transformer(2017 "Attention Is All You Need")로 진화했고, self-attention + positional encoding이 병렬 처리와 long-range dependency를 가능케 했다.
- Transformer는 BERT(bidirectional encoder; 이해·분류)와 GPT(unidirectional decoder; 생성) 두 계열을 낳았고, 대규모 unlabeled pre-training → fine-tuning 패러다임을 정착시켰다.
- scale 확대(GPT-1 117M → GPT-3 175B → DeepSeek R1 671B)가 few-shot/zero-shot 능력을 unlock하며 "LLM" 용어가 등장. 이 책에서 LLM/Transformer는 decoder-only를 지칭.

---

## 2. The Autoregressive Nature of Transformers
> LLM은 한 번에 token 하나씩 생성하며, 새 token을 입력 sequence에 append해 다음 token 예측의 입력으로 사용하는 autoregressive 방식이다.
- 예: "Write a short introduction about the US capital city" → Washington → D.C. → is → the … (max length 또는 stop token까지 반복).

---

## 3. Decoder-Only Transformer Architecture
> decoder-only 모델은 tokenizer+embedding → 다수의 Transformer(decoder) block → LM head 세 컴포넌트로 구성된다.
- tokenizer/embedding: 텍스트를 token → token ID → embedding vector로 변환.
- decoder blocks(12·24·… 층): 대부분의 연산 발생, 출력은 hidden state([N, d] tensor; d=768·2048·4096 등).
- LM head: hidden state → vocabulary logits(확률분포) → 최고 확률 token 선택.
- 실무 권장: 배포 전 model config(layer 수·hidden size·attention head·vocab size) 검사 → GPU memory·quantization·sharding 전략 결정. 예 Qwen2.5-0.5B: hidden 896, 24 layers, 14 heads, vocab 151,936, ~494M params.

---

## 4. Transformer (Decoder) Block
> 각 decoder block은 self-attention layer(token 간 관계·context 반영)와 feedforward network(FFN; per-token 표현 정제) 두 핵심 요소로 구성된다.
- Qwen2.5 구조 예: self_attn(q/k/v/o_proj) + mlp(gate/up/down_proj, SiLU) + RMSNorm.
- attention-heavy block은 fused attention kernel·custom CUDA로 최적화 여지가 있다.

---

## 5. Capture Token Context by Calculating Attention
> self-attention은 각 token마다 query(Q)·key(K)·value(V)를 계산하고, Q·K dot product → scale → softmax → V 가중합으로 context가 반영된 token 표현을 만든다 (Attention = softmax(QK^t/√dk)·V).
- multi-head attention: 한 token에 대해 여러 head가 각자 Q/K/V projection으로 병렬 계산 → syntactic·positional·semantic 관계를 동시에 포착 → concat 후 linear layer.
- serving 관점 핵심: attention은 compute-intensive(특히 prefill)이며 memory·latency가 sequence length에 비례 — 수학적 디테일보다 이 직관이 KV caching·GPU 분산에 중요.

---

## 6. Executing LLM Generation — KV Cache 없이
> AutoModelForCausalLM으로 token을 하나씩 생성하는 loop(model(idx_cond) → logits → softmax → multinomial sample → append)로 LLM 실행을 직접 구현해 serving-oriented mental model을 만든다.
- 데모: 100 token 생성에 9.12초(~0.09초/token). 첫 token 이후 per-token 시간이 점점 증가 — 매 step마다 길어진 전체 sequence의 attention을 재계산하는 비효율 발생.

---

## 7. Enable the KV Cache to Boost Performance
> KV cache는 이전 token들의 attention key/value vector를 layer별로 저장해, 새 token 생성 시 신규 token만 처리하고 중복 계산을 건너뛴다 (memory ↑ 대신 compute 대폭 절감).
- 구현 핵심: input을 신규 token만 전달(input_ids = generated_token_id) + past_key_values 전달 + use_cache=True + 매 step past_key_values 갱신.
- 데모: 동일 100 token 생성이 9.12초 → 3.14초로 단축. 첫 token 이후 안정적·빠른 속도.

---

## 8. The Prefill and Decode Phases
> prefill phase는 전체 prompt를 한 번에 처리(병렬, compute-intensive, sequence length에 대해 quadratic), decode phase는 token을 하나씩 생성(memory-intensive, weight 로드·KV cache 증가).
- bottleneck 진단: 긴 prompt(500+ page PDF)는 prefill이 비싸고, 짧은 prompt·긴 생성(chatbot·story)은 decode가 bottleneck.
- 첫 token(prefill) 시간이 가장 길고, 이후 decode token들은 짧고 일정 — phase별 최적화 전략은 ch6~7에서 다룸.

---

## 9. Run the LLM with a Serving Framework (vLLM)
> serving framework(vLLM·SGLang)는 pre-trained 모델을 로드해 API로 노출하며 KV-cache 재사용·request scheduling(batching)·multi-user concurrency·token streaming/cancellation·최신 최적화(paged attention·speculative decoding)를 제공한다.
- vLLM 기본 사용: LLM(model=...) + SamplingParams + llm.generate()로 몇 줄에 inference 가능. 고급 config: swap_space, block_size, enable_prefix_caching, enable_chunked_prefill, enable_cuda_graph, tensor parallel 등.
- 성능: 동일 prompt에서 vLLM 1.12초 vs HF 19.58초 (~17배). best practice는 HF로 prototype 후 vLLM으로 production 이전·튜닝.

---

## 10. LLM Streaming Serving Basics
> streaming은 전체 출력 완료를 기다리지 않고 생성되는 대로 token을 즉시 반환하는 기법으로, chatbot·live captioning 등 실시간 UX에 필수다.
- vLLM 구현: AsyncLLMEngine + AsyncEngineArgs로 async stream(AsyncStream) 반환 → "async for request_output in results_generator"로 token을 하나씩 pull.
- 부가 이점: engine.abort(request_id)로 원치 않는 생성을 중간 취소 → UX 개선 + compute 자원 절약.

---

## 11. LLM Batch Serving Basics
> batching은 여러 input 요청을 묶어 단일 forward pass로 동시 처리해 GPU 자원을 충분히 활용하고 throughput을 높이는 기법이다.
- Transformer는 matrix multiplication·attention이 sequence 간 병렬화 가능 + weight 공유로 batching 효율이 높다.
- 데모: 4 prompt batch 1.06초 vs 하나씩 2.39초 (~2.2배). continuous batching(신규 요청 동적 추가, ch6)은 Anyscale 2023 연구 기준 최대 23배 throughput + p50 latency 감소.

---

## Summary (핵심 정리)
- decoder-only Transformer는 tokenizer+embedding → decoder blocks(self-attention+FFN) → LM head로 구성되며 autoregressive하게 token을 하나씩 생성한다.
- KV cache는 이전 token의 key/value를 재사용해 중복 attention 계산을 제거(예: 9.12→3.14초), prefill(compute-intensive)/decode(memory-intensive) phase 구분은 bottleneck 진단·최적화의 기반이다.
- vLLM 같은 serving framework는 저수준 복잡성을 추상화하면서 HF 대비 10~20배 성능을 제공하고, KV-cache 재사용·scheduling·concurrency·streaming을 내장한다.
- streaming(token 즉시 반환·중간 취소)과 batching(동시 처리로 throughput 향상, continuous batching으로 최대 23배)은 production-grade LLM serving의 핵심 전략이다.
