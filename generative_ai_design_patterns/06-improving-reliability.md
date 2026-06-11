# 06. Improving Reliability

## 챕터 개요 (3줄 요약)
- foundational model은 본질적으로 stochastic → 일관성 없는 출력·factual 오류·hallucination 문제. 이를 완화하는 4개 패턴(17~20)을 다룬다.
- LLM-as-Judge(평가), Reflection(자기 비판·반복 개선), Dependency Injection(테스트 가능 모듈화), Prompt Optimization(prompt 자동 최적화)으로 신뢰성 확보.
- 평가·자기교정·모듈 설계·입력 최적화를 통합해 production 신뢰도와 사용자 confidence를 높인다.

---

## 1. Pattern 17: LLM-as-Judge
> LLM으로 출력을 평가해 다차원 피드백 제공 — 완전 자동 metric과 human 평가 사이의 확장 가능하고 nuanced한 중간지대.

- **문제**: open-ended task 평가의 어려움 — outcome measurement(혼재 변수), human evaluation(고비용·bias·확장 불가), automated metric(BLEU/ROUGE는 의미·정확성 못 잡음).
- **Option 1 — prompting**: 커스텀 scoring rubric을 LLM에 적용. temperature 0, 입력 self-contained화, calibration rubric 구체화(점수별 기준 명시). 단일/pairwise/ranking 선택.
- **Option 2 — ML**: rubric 점수 + 실제 outcome(CRM 구매 데이터)으로 classification 모델 학습 → 무관 기준 자동 discount.
- **Option 3 — fine-tuning**: human expert 점수로 Adapter Tuning(의료 진단 체크리스트처럼 인간 모방).
- **고려사항 — inconsistency**: coarse score(1~5, binary 최선), multiple criteria(=CoT), multiple evaluation(LLM-as-jury, polling).
- **leniency**: LLM은 후한 채점(A/B만 줌) → 직접 비교가 점수 비교보다 나음, GRPO group reward, 기대치 하향(문제 식별용).
- **bias**: self-bias(자기 출력 선호 → 다른 LLM로 평가), length bias·positional bias. 소형 fine-tuned(PandaLM)·PatronusAI 고려.
- **caveat**: justification 요구는 interpretability 주나 평가 성능·bias에 부정적일 수 있음.

## 2. Pattern 18: Reflection
> AI 시스템이 최종화 전 자기 reasoning·출력을 평가·비판해 오류를 잡고 반복적으로 개선하는 agentic 패턴.

- **문제**: API 호출은 stateless → 어떻게 자동으로 critique 생성하고 이전 응답을 교정하나.
- **해법**: LLM을 2회+ 호출 — 첫 응답을 evaluator(LLM/tool/human)에게 보내 단순 점수가 아닌 **critique**(어떻게 부족한지) 받음 → 피드백으로 prompt 수정 → 재생성 → 품질 기준 충족까지 반복.
- **변형**: maximum attempts(무한루프 방지, 1회 retry면 threshold 불필요 — Zero-shot CoT "reconsider"), conversational state(critique를 메시지 이력으로 추가, Autogen).
- **예시**: 로고 디자인 — Gemini로 생성, Claude로 LLM-as-Judge critique, 1라운드 reflection으로 개선(leniency 때문에 threshold 어려워 1회 critique가 실용적).
- **고려사항**: Ng의 4 agentic pattern 중 하나. 품질·robustness·transparency 향상. cost vs quality trade-off(code gen은 다단계 reflection 유익, chatbot/realtime은 latency 부담) — reflection depth를 문제·시간·비즈니스 영향에 맞춰 조정. evaluation이 핵심 — self-bias 회피 위해 다른 LLM 사용.

## 3. Pattern 19: Dependency Injection
> LLM chain의 각 컴포넌트를 독립 개발·테스트할 수 있도록, 어떤 step이든 mock 구현으로 교체 가능하게 만든다.

- **문제**: GenAI 앱 개발·테스트 어려움 — nondeterministic(같은 입력에 다른 출력), 모델 빠르게 변함(prompt brittle), LLM-agnostic 필요(여러 모델 테스트). 특히 chain에서 한 step 출력이 다음 입력 context가 될 때.
- **예시**: 마케팅 설명 개선 — Step 1(critique 생성, structured Critique) + Step 2(suggestion 구현, Improvement). assert로 테스트(improvements>3, 변경 라인 1~5, 선택된 change가 원래 목록에). 문제: Step 2 테스트하려면 Step 1을 호출해 Critique를 얻어야 함.
- **해법**: chain을 동일 signature의 다른 구현으로 교체 가능하게 정의(`critique_fn`, `improve_fn` 주입). 개발·테스트 시 hardcoded 값 반환하는 `mock_critique` 주입. Python -O로 assertion on/off, Pytest로 상세 정보.
- **고려사항**: 함수뿐 아니라 abstract class·inheritance로 객체도 mock 가능. 상호작용 복잡해지면 hardcoded mock 값 맞추기 어려움. 외부 함수도 mock해 network latency·가용성 의존 제거. SW 엔지니어링의 오랜 패턴이나 GenAI 프레임워크는 native 미지원(Pydantic이 근접).

## 4. Pattern 20: Prompt Optimization
> dependency 변경 시 prompt를 예시 dataset에 대해 최적화해 체계적으로 갱신한다 ("another level of indirection").

- **문제**: prompt engineering은 trial-and-error → 모델 버전·toolchain 변경 시 모든 trial 반복 필요(brittle).
- **4 구성요소**: ① pipeline of steps(프레임워크가 prompt 주입), ② dataset(input±reference, 1개~수천), ③ evaluator(reference 비교 또는 LLM-as-Judge fitness), ④ optimizer(prompt 변형 생성·평가·최적 pipeline 반환).
- **예시 (DSPy)**: O'Reilly 책 뒷표지 blurb 개선 — BlurbExtraction(text→Blurb) + BlurbImprovement signature를 Module로 결합, `forward()`가 순차 호출. prompt는 DSPy가 자동 생성(reasoning 필드 추가 = CoT).
- **평가**: LLM-as-Judge로 원본 reference 대비 -1~1 점수, 길이 penalty 후 aggregate.
- **optimizer**: `BestOfN`(N=10 변형 중 최선, threshold; inference 10회) vs `BootstrapFewShot`(예시 3개 선택·나머지 7개 평가 반복 → 여러 책에 통하는 prompt, inference 1회). dataset 크면 LLM fine-tune도 가능.
- **고려사항**: prompt library(외부화)는 수동 실험 문제를 못 풂. Prompt Optimization은 코드에 prompt가 없어 모델 버전 변경 시 rerun만으로 갱신. 다른 프레임워크: AdalFlow, PromptWizard. 기록된 prompt·feedback으로 LLM-as-Judge·post-training 데이터셋 구축 가능.

---

## Summary (핵심 정리)
- foundational model의 stochastic 특성으로 인한 비일관성·오류·hallucination을 4개 패턴으로 완화.
- **LLM-as-Judge**: 커스텀 rubric으로 출력 평가(prompting/ML/fine-tuning), inconsistency·leniency·bias 주의(다른 LLM·binary·jury).
- **Reflection**: critique 생성 → prompt 수정 → 재생성 반복, code gen에 유익하나 latency trade-off.
- **Dependency Injection**: 동일 signature mock 주입으로 chain step 독립 개발·테스트.
- **Prompt Optimization**: pipeline·dataset·evaluator·optimizer로 prompt를 자동 최적화(DSPy), dependency 변경에 강건.
- LLM-as-Judge와 Reflection은 출력 품질, Dependency Injection은 구조적 신뢰성, Prompt Optimization은 일관성 담당.