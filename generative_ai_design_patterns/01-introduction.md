# 01. Introduction

## 챕터 개요 (3줄 요약)
- GenAI 프로토타입은 쉽지만 production은 어렵다 — hallucination, nondeterminism, 학습에서 비롯된 한계 때문. 이 책은 그런 반복적 문제에 대한 32개 design pattern을 카탈로그화한다.
- foundational model 위에 application을 쌓는 접근을 AI engineering이라 부르며, prompt/context, 모델 생성 과정, 샘플링 제어, in-context learning, post-training 등 공통 기반 개념을 정리한다.
- agent란 foundational model을 두뇌로 삼아 autonomy·goal orientation·planning·perception/action을 갖춘 software component이며, 이 패턴들은 더 나은 agentic application 구축을 돕는다.

---

## 1. GenAI Design Patterns
> design pattern은 반복 문제에 대한 검증된 해법이자 공통 vocabulary로, software 품질·유지보수성·확장성을 높인다.

- Christopher Alexander의 건축 pattern → GoF의 *Design Patterns* 로 software engineering에 정착.
- 오늘날 개발자는 custom ML 모델을 from scratch로 학습하기보다 GPT/Gemini/Claude/Llama 등 foundational model을 활용 (= AI engineering, 실무자 = AI engineer).
- prompt만 보내 콘텐츠를 생성하지만 style 불일치·enterprise 지식 부재·capability 부족 같은 공통 문제 발생 → 이 책의 패턴들이 해법.

## 2. Building on Foundational Models
> hosted foundational model을 API로 호출하며, model/framework/hyperscaler에 agnostic하게 접근한다.

- **Prompt와 Context**: 단순 prompt는 instruction, 복잡한 prompt는 context(역할·정보)까지 포함. prompt/response 모두 multimodal 가능.
- **Model Provider API**: Anthropic API에서 `system` prompt(개발자가 고정한 전체 행동)와 `user` prompt(동적 task 지시)를 분리.
- **LLM-Agnostic Framework**: PydanticAI의 `Agent`는 model 문자열만 바꿔 provider 전환 (`anthropic:...`, `openai:gpt-4o-mini` 등).
- **Local 실행**: Ollama로 open-weights 모델을 받아 OpenAI 호환 API로 호출.

```python
from pydantic_ai import Agent
agent = Agent('anthropic:claude-3-7-sonnet-latest',
    system_prompt="You are an expert Python programmer.")
result = agent.run_sync("Write code to find the median value of a list of integers.")
```

## 3. How Foundational Models Are Created
> foundational model은 pretraining → SFT → RLHF의 다단계로 만들어지며, 내부 vocabulary 이해용으로만 알면 된다.

- **Pretraining**: 대규모 token corpus(DeepSeek: 14.8조 token)로 next-token prediction 학습 → "next-token predictor".
- **SFT**: 사람이 작성한 (prompt, response) 예시로 instruction following 개선. DeepSeek-V3는 MoE(671B 중 token당 37B만 활성).
- **RLHF / preference optimization**: 사람 선호 쌍으로 정렬.
- DeepSeek-R1은 cold start → pure RL → rejection sampling → SFT → final RL의 multistage 과정. 핵심 돌파구는 SFT 없이 pure RL만으로 reasoning(CoT)을 유도한 것.
- distillation으로 Qwen/Llama 기반 소형 버전(1.5B~14B) 제공.

## 4. The Landscape of Foundational Models
> 모델은 frontier, distilled, open-weight, locally hostable로 분류되며 LMArena의 blind pairwise(Elo) 비교가 사실상 표준 평가다.

| 분류 | 특징 | 예시 |
|---|---|---|
| Frontier | SOTA·고비용·local 불가 | GPT-5, Gemini 2.5 Pro, Claude |
| Distilled | 성능/효율 균형, 저비용·고속 | Gemini Flash, Claude Sonnet, GPT-4o-mini |
| Open-weight | parameter 공개, fine-tune 가능, hosting에 전문성 필요 | Llama, Mistral, DeepSeek, Qwen, Falcon |
| Locally hostable | edge/air-gapped, privacy, 능력 제한 | Llama 8B, Gemma 2B |

## 5. Agentic AI
> agent는 foundational model을 두뇌로 삼아 자율적으로 목표를 달성하는 software component이다.

- **Autonomy**: "어떻게"가 아닌 "목표"만 주면 됨 (예: just-in-time 재고 관리 agent).
- **Agent 특성**: goal orientation, planning & reasoning, perception & action(Tool Calling으로 외부 함수 호출), adaptability & learning(Reflection·Self-Check로 self-correct).
- 현재 agentic 동작은 aspirational — nondeterminism·hallucination이 완전 자율을 가로막음. 이 책의 다수 패턴이 application을 더 agentic하게 만드는 방법.

## 6. Fine-Grained Control
> logits → softmax 변환과 sampling 전략을 이해하면 패턴 없이도 생성 과정을 세밀히 제어할 수 있다.

- **Logits**: 마지막 layer의 unnormalized 출력 → softmax로 확률화. greedy sampling(최댓값만)은 반복적·지루한 텍스트 유발.
- **Temperature(T)**: logits를 T로 나눠 randomness 제어. T=0이면 greedy, 높을수록 tail word 선택 ↑ (창의성 ↑). RAG·LLM-as-Judge에선 낮은/0 temperature 사용.
- **Top-K sampling**: 상위 k개 token만 고려해 long tail 절단.
- **Nucleus(Top-P) sampling**: 누적 확률이 p를 넘는 최소 집합 선택, 모델 confidence에 적응 → 더 자연스러움.
- **Beam search**: 여러 continuation을 병렬 탐색해 전체 sequence 확률 최적화. frequency/presence penalty(반복 억제), length penalty, beam width 제어.

```
                  ┌─────────┐   logit_i / T
   raw logits ──▶ │ scale T │ ──────────────┐
                  └─────────┘                ▼
                                       ┌──────────┐
   top-K / top-P filter ──────────────▶│ softmax  │──▶ P(token_i)
                                       └──────────┘
```

## 7. In-Context Learning
> weight 변경 없이 prompt 안의 예시·지시만으로 새 task를 수행하는 능력으로, 빠른 prototyping과 production 업데이트에 유리하다.

- **Zero-shot**: 예시 없이 지시만으로 수행 (pretrained 지식 + 자연어 이해 의존).
- **Few-shot**: 소수 예시를 prompt에 포함 → 구조·출력 형식을 학습. context engineering의 단순·효과적 형태.
- **장점**: dataset curation 불필요, 빠른 prototyping, 빠른 update.
- **한계**: 모델에 이미 지식·능력이 있어야 함; 예시가 context window token 소모·inference 지연; 복잡 문제 일반화 어려움 → 이 경우 post-training 고려.

## 8. Post-Training
> pretrained 모델의 weight를 수정해 새 task·domain에 맞추는 방법으로, 별도 endpoint로 배포된다.

- **CPT(Continued Pretraining)**: 새 vocabulary/연관 학습, full weights 필요·고비용 (Bloomberg 사례 후 거의 채택 안 됨).
- **SFT / instruction tuning**: (prompt, response) 쌍으로 추가 학습. 단일 task만 하면 catastrophic forgetting, 다양한 task면 일반화.
- **PeFT**: LoRA(원 weight freeze + 저랭크 adapter, trainable param 최대 1만배 ↓, latency 추가 없음), QLoRA(weight quantize, memory 효율).
- **Preference tuning**: RLHF, 효율적인 DPO, DeepSeek의 GRPO(group 정규화 reward).
- **Fine-tuning 고려사항**: 100+ 샘플 데이터 요구, catastrophic forgetting, 추가 복잡성(평가·재학습·lineage 추적), 추가 비용(hosted는 per-token ↑, open은 GPU 비용).

## 9. The Organization of the Rest of the Book
> 이 책은 8개 챕터에 걸쳐 32개 패턴을 다루고, 마지막에 이를 조합한 production agentic application을 보여준다.

- Ch2 Controlling Content Style (Pattern 1-5), Ch3-4 Adding Knowledge/RAG (Pattern 6-12), Ch5 Extending Capabilities (Pattern 13-16), Ch6 Improving Reliability (Pattern 17-20), Ch7 Enabling Agents to Act (Pattern 21-23), Ch8 Addressing Constraints (Pattern 24-28), Ch9 Setting Safeguards (Pattern 29-32), Ch10 Composable Agentic Workflows.

---

## Summary (핵심 정리)
- GenAI는 prototyping은 쉽지만 production은 hallucination·nondeterminism 때문에 어렵다 → 32개 design pattern이 반복 문제의 검증된 해법.
- AI engineering = foundational model 위에 application을 쌓는 접근, agent = 모델을 두뇌로 한 autonomous software component.
- foundational model 생성: pretraining(next-token) → SFT → RLHF, DeepSeek-R1은 pure RL로 reasoning 유도.
- 모델 분류: frontier / distilled / open-weight / locally hostable, 평가는 LMArena Elo blind 비교.
- 생성 제어: temperature, top-K, nucleus(top-P), beam search + repetition/length penalty.
- 새 task 적응: prompt만 바꾸는 in-context learning(zero/few-shot) vs weight 수정하는 post-training(CPT/SFT/PeFT-LoRA/preference tuning).