# 05. Extending Model Capabilities

## 챕터 개요 (3줄 요약)
- LLM은 패턴 인식·next-token 예측엔 뛰어나나 training data에 없는 esoteric·industry-specific task(투자위원회 메모, 내부 조사)는 못 한다 — 이를 가르치는 4개 패턴을 다룬다.
- prompt 측 해법(CoT의 reasoning, ToT의 path 탐색)과 weight 측 해법(Adapter Tuning, Evol-Instruct instruction tuning)으로 나뉜다.
- math/reasoning 예시는 한계를 보이기 위한 것일 뿐, 실제 패턴은 도메인 특화 task 해결용.

---

## 1. The Limits of LLM Reasoning
> LLM은 훈련 데이터의 manipulation을 일반화할 뿐 semantic·logical 관계를 인간처럼 이해하지 못한다.

- **known**: 100~110 소수, 84㎡→ft² 변환 등은 일반화로 풀림 (단 곱셈 결과는 hallucinate — calculator tool 필요).
- **unknown**: bridge "eight ever, nine never" maxim은 알지만 AKJxx 실제 적용은 틀림 → 격언 재생산 ≠ 적용.
- model provider가 보고된 오류를 hardcode·재훈련으로 수시 수정하므로 특정 예시는 재현 안 될 수 있으나 현상은 지속.

## 2. Pattern 13: Chain of Thought
> 직접 답 대신 중간 reasoning step을 거치도록 prompt해 multistep 문제 해결력을 높인다.

- **문제**: training data coverage 부족(grade school 물리는 OK, oil&gas flow rate는 "lazy" 거부), multistep reasoning 비약, black-box 답(사후 "why"는 hallucinate).
- **Zero-shot CoT**: "think step-by-step" 추가 → pretrained 능력 unlock. 소형/local 모델에 특히 유효.
- **Few-shot CoT**: 유사 문제의 step-by-step 예시 제공 → 모델이 일반화. RAG는 "고기를 줌(data)", CoT는 "낚시를 가르침(logic)". pretrained에 없는 logic도 가능(baggage·bridge 문제 해결).
- **Auto-CoT**: example store(vector/doc DB)에 Zero-shot CoT로 생성·검증한 예시 축적, 새 질문에 유사 5개 동적 선택.
- **한계**: data gap(Hyderabad 서쪽 300km → Nanded hallucinate, map 추가로 Solapur 교정), nonsequential logic(bridge expert line은 mimic 불가).
- **대안**: thinking mode(test-time compute) 내장 모델, agentic(Tool Calling·ReAct), ToT. CoT는 6개월마다 여전히 필요한지 재검토 권장.

```
CoT:  prompt + "think step-by-step" ──▶ step1 ─▶ step2 ─▶ ... ─▶ answer  (linear)
ToT:  prompt ──▶ {thought a, b, c} ─▶ evaluate ─▶ beam(top-K) ─▶ ... ─▶ backtrack ─▶ summary  (tree)
```

## 3. Pattern 14: Tree of Thoughts (ToT)
> 문제 해결을 tree search로 다뤄 여러 reasoning path를 생성·평가·탐색하고 실패 시 backtrack한다 — CoT의 linear 한계를 넘는다.

- **문제**: 전략적·논리적 task는 단일 path로 안 풀림(4문장 essay 문제 — 초기 path 고착, 단일 reasoning, 중간 평가 없음).
- **4 구성요소**: thought generation(다양한 다음 step JSON 생성), path evaluation(0~100 점수), beam search(top-K 유지), summary generation(최선 path로 최종 답).
- **tree search**: heapq.nsmallest로 beam 효율 선택, score>0.9면 조기 종료, BFS/DFS도 가능.
- **예시**: supply chain 최적화(3 manufacturing × 4 DC × 2 shipping, 시나리오별 평가) — depth 4, 41 API call, 93초로 Configuration C(resilience) 도출.
- **고려사항**: combinatorial explosion, 높은 latency·비용(수백 call), 구현 복잡성(state·path 추적).
- **대안**: reasoning model(o3·Opus·Gemini 2.5·R1), least-to-most prompting, Reflection, wait-injection(종료 토큰을 "Wait"로 덮어 재평가). Ng의 4 agentic pattern(Reflection·Tool Use·Planning·Multiagent) 중 ToT는 4개 모두 해당.

## 4. Pattern 15: Adapter Tuning
> 소량의 add-on layer만 효율적으로 학습해 foundational model을 특정 task에 fine-tune한다 (PeFT/LoRA·QLoRA).

- **문제**: 수백 input-output 쌍이 있을 때 prompt engineering(확장성·비용·테스트 어려움)도 few-shot(context 소모·확장 불가)도 부적합.
- **아키텍처**: transformer block에 adapter layer 삽입(dim 축소 dense → ReLU → dim 복원 dense), foundational weight는 freeze. r=16 등 저랭크 → QLoRA(quantized).
- **용도/금지**: classification·summarization·extractive QA·brand chatbot 등 "기존 능력에 가까운 특화 task"에만. **industry jargon·새 언어(CPT 필요)·새 knowledge(RAG 필요)·새 task family(Evol-Instruct 필요)에는 부적합.**
- **학습**: TRL/Unsloth로 4-bit 모델 로드 → LoraConfig(target_modules, lora_alpha) → SFTConfig(낮은 learning rate, epoch) → SFTTrainer. 100~수천 예시, 단일 GPU 1시간 이내.
- **예시**: Gemma 3 4B multimodal로 radiology 이미지 captioning — 500 image-text 쌍, loss 14.8→4.0, 간결한 해부학 caption 생성.
- **변형/대안**: merged model(배포 단순·latency↓) vs 분리 저장(storage↓); closed-weights는 provider managed fine-tune(Vertex AI). 단순하면 few-shot·CoT·Content Optimization 고려.

## 5. Pattern 16: Evol-Instruct
> 초기 instruction을 evolve해 대규모 dataset을 만들고 instruction tuning(SFT)으로 pretrained 모델에 새롭고 복잡한 task를 가르친다.

- **문제**: enterprise task(창고 적합성 보고서 등)는 provider가 알지도, 학습 데이터(기밀)도 없음 → data privacy 정책상 모델이 자동 개선 안 됨.
- **4단계**: ① instruction evolve(deepen·concretize·add constraint·combine), ② answer 생성(human expert / industry tool / reflection 평가루프 / RAG / teacher-student), ③ evaluate&filter(LLM-as-Judge로 고품질만, "Textbooks Are All You Need"), ④ instruction tuning(SFT).
- **instruction tuning**: Adapter Tuning(수백 예시)보다 훨씬 큰 수천~만 단위 dataset 필요. PeFT 시 gate_proj·embed_tokens·lm_head까지 튜닝 + embedding/linear 학습률 분리(Unsloth).
- **예시**: Gemma 3 1B를 S&P 500 business strategy 컨설턴트로 — SEC EDGAR item_7(management discussion)에서 질문 bootstrap → 13문항/filing × 2000 filing → Gemini로 답 생성·평가 → 점수 4~5만, ~11K 예시 3 epoch(L4 3시간). 결과가 frontier 모델(Claude)보다 나음.
- **고려사항**: dataset 크기 ∝ task 복잡도, 반비례 ∝ 모델 크기(1B는 ~10K, 10B는 ~1K). catastrophic forgetting 예상 → 좁은 task에만 사용. dataset 생성에 예시당 3+ LLM call(10K → 30K+ call)로 고비용.

---

## Summary (핵심 정리)
- LLM은 training data에 없는 task는 못 함 → CoT·ToT(prompt 측), Adapter Tuning·Evol-Instruct(weight 측)로 확장.
- **CoT**: 중간 reasoning step 유도(Zero/Few-shot/Auto-CoT), data gap·nonsequential logic엔 한계.
- **ToT**: tree search로 다중 path 탐색·backtrack, 전략·계획 task에 강하나 고비용·복잡.
- **Adapter Tuning**: add-on layer만 학습(LoRA/QLoRA), 기존 능력에 가까운 특화 task용 — jargon·새 지식엔 부적합.
- **Evol-Instruct**: instruction evolve로 대규모 dataset 생성 → instruction tuning으로 새 enterprise task 학습.
- 선택 기준: 단순→few-shot/CoT, frontier가 잘하면 Adapter, 복잡·frontier 실패 시 Evol-Instruct.