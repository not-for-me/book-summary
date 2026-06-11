# 02. Controlling Content Style

## 챕터 개요 (3줄 요약)
- foundational model 출력은 stochastic해서 같은 prompt도 model·시도마다 style이 크게 달라진다 — branding·accuracy·compliance·format을 위해 style을 제어하는 5개 pattern을 제시.
- 제어 강도 순: Logits Masking·Grammar(엄격 강제) → Style Transfer·Reverse Neutralization(예시 기반 암묵적) → Content Optimization(최적 style 자동 탐색).
- 핵심 antipattern은 "try-and-try-again"(생성→검증→재생성)과 prompt로 format 준수를 "구걸"하는 것; 둘 다 brittle·고비용·non-convergent.

---

## 1. Pattern 1: Logits Masking
> sampling 단계를 가로채 규칙에 맞지 않는 continuation의 logit을 -inf로 zero out해 생성 텍스트가 규칙을 강제 준수하게 한다.

- **문제**: branding(brand word 사용), accuracy(invoice ID 중복 금지), compliance(경쟁사 언급 금지), stylebook 준수 등 규칙 강제.
- **antipattern (try-and-try-again)**: geometric distribution — 성공률 p%면 평균 100/p회 생성. p=90이면 99분위 2회, p=30이면 13회 → 성공률 매우 높을 때만 허용.
- **구현 (Transformers)**: `LogitsProcessor` subclass의 `__call__` 오버라이드, token decode 후 `apply_rules` 불만족 시 `output_logits[idx]=-np.inf`. backtrack/regeneration은 `pipe.model.generate()`를 16 token씩 직접 제어.
- **예시**: sequence selection(영양제 ad — positive 키워드 최대화, banned word 회피), sequence regeneration(acrostic 시 — 첫 글자가 단어를 철자화하도록 backtrack).
- **caveats**: logprob 접근 필요(Anthropic 미지원, OpenAI/Google/Meta 지원), client-model 통신으로 latency↑(주로 local model에 적합), 후보 없으면 refuse. RL의 invalid action masking에서 유래.

## 2. Pattern 2: Grammar
> context-free metasyntax(BNF) 또는 schema로 token 생성을 제약해 출력이 특정 format/data schema에 정확히 부합하게 한다 (= 모델 측 Logits Masking).

- **antipattern**: "provide output in JSON" 같이 instruction-following에 의존 → brittle·unreliable·costly.
- **Option 1 — BNF grammar**: `IncrementalGrammarConstraint` + `GrammarConstrainedLogitsProcessor`로 SQL timestamp, CSV, regex 등 강제 (client-side).
- **Option 2 — standard format**: OpenAI `response_format={"type":"json_object"}` (prompt에 JSON 명시 필요).
- **Option 3 — schema/dataclass(structured output)**: Python `@dataclass`/Pydantic, `response_schema=Receipt` → server-side Logits Masking, network call↓.
- **BNF vs Pydantic**: BNF는 logic·validation·동적 규칙(credit card 형식 등)에 유연하나 logprob 필요·복잡; Pydantic은 ease of use·낮은 latency·범용 지원이나 Enum 넘어선 validation은 어려움.
- **caveats**: 후보 token 없으면 endless whitespace·refusal↑·부정확. 별칭: structured outputs, constrained decoding (단, LangGraph의 추가 LLM call 방식은 진짜 Grammar pattern 아님).

```
prompt ──▶ LLM ──▶ candidate tokens ──▶ [Grammar/BNF filter] ──▶ valid token only ──▶ output
                                          (zero out disallowed)
```

## 3. Pattern 3: Style Transfer
> input-output 예시 쌍을 보여줘 가용 콘텐츠를 원하는 tone/style로 변환하도록 가르친다 (few-shot 또는 fine-tuning).

- **적용 조건**: ① content는 있으나 원하는 style이 아님, ② nuance를 규칙으로 표현 어려움("I know it when I see it"), ③ 전문가가 변환한 예시 보유.
- **Option 1 — few-shot**: prompt에 1~10개 input/output 예시 삽입 (teacher-student).
- **Option 2 — fine-tuning**: 100~수천 예시로 학습 → 높은 fidelity, prompt 축소로 빠르고 저렴한 inference. 단점: data curation·training cost·전문성·LLMOps·catastrophic forgetting.
- **이미지 적용**: Stable Diffusion + ControlNet으로 depth map(control image)을 써 공간 구조 보존하며 style transfer (Wanderer 그림 → Star Wars).
- **고려사항**: 큰 모델일수록 일반화↑; context 한계·예시 과다 시 "혼란"·inference 지연. 엄격 강제는 보장 못 함.

## 4. Pattern 4: Reverse Neutralization
> 중간 neutral form을 거쳐 style을 입힌다 — foundational model이 neutral 콘텐츠 생성, fine-tuned model이 neutral→원하는 style로 변환.

- **차별점**: Style Transfer와 달리 handcrafted input-output 쌍이 없어도 됨 (style 예시만 있으면 됨).
- **dataset 생성 3단계**: ① 보유한 styled 글을 LLM으로 neutral화, ② input/output을 뒤집어(neutral→styled) training pair 생성, ③ base model을 fine-tune.
- **inference 2단계**: foundational model로 neutral 콘텐츠 생성 → fine-tuned model로 personal/legal style 변환.
- **예시**: 인도 Tamil Nadu legalese 편지 생성, 개인 문체 이메일 생성(over-the-top 예시로 효과 시연).
- **고려사항**: neutral form이 repeatable해야(LLM마다 "neutral" 정의 다름); embedding cosine similarity로 의미 보존 검증; over-neutralization(과도하면 의미 손실) 주의. back translation과 유사.

## 5. Pattern 5: Content Optimization
> 어떤 style 요인이 중요한지 몰라도, pairwise 비교로 winner/loser dataset을 만들고 preference tuning(DPO)으로 최적 style을 내는 모델을 학습한다.

- **문제 (A/B 한계)**: style 요인 가설이 없으면 set 구분 불가·통계적 유의성 도달 불가·결과 활용 불가.
- **해법 재정의**: 두 콘텐츠만 비교해 winner=Set A; test=단일 비교(유의성 불필요); prompt 대신 LLM weight를 바꿈.
- **4단계**: ① 같은 prompt로 콘텐츠 쌍 생성(repeated generation / 설정 변경 / prompt rewriting), ② 우열 선정(human labeling / evaluator·LLM-as-judge / 실제 outcome), ③ {prompt, chosen, rejected} dataset 생성, ④ DPO(`DPOTrainer`, TRL) tuning.
- **핵심**: Step 2(evaluation)가 가장 중요 — metric이 objective의 robust한 proxy여야(engagement time이 난해함을 유발하지 않도록). "metric ≠ objective".
- **고려사항**: in-distribution 요구(생성 콘텐츠를 tuning 대상 모델이 만들 수 있어야 — 같은 모델 사용 또는 SFT 선행); 이미지에도 DiffusionDPO 적용; 빠른 평가 시 iterative training으로 saturation까지 지속 개선.

---

## Summary (핵심 정리)
- foundational model 출력은 stochastic → branding/accuracy/compliance/format을 위해 style 제어가 필요.
- **Logits Masking**: sampling 가로채 비준수 logit zero out, 동적 규칙 강제(logprob 필요).
- **Grammar**: BNF/schema로 token 제약 = 모델 측 Logits Masking, structured output/constrained decoding (Pydantic이 가장 범용).
- **Style Transfer**: 예시 쌍으로 style 변환(few-shot/fine-tuning), nuance를 규칙화 어려울 때.
- **Reverse Neutralization**: neutral 중간형 경유, handcrafted 쌍 없이도 styled 콘텐츠 생성.
- **Content Optimization**: 요인 몰라도 pairwise 비교 + DPO preference tuning으로 최적 style 학습.
- 공통 antipattern: try-and-try-again, prompt로 format 준수 구걸.