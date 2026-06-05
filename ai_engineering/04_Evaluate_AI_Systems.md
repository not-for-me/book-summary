# 04. Evaluate AI Systems

## 챕터 개요 (3줄 요약)
- 모델은 의도한 애플리케이션 맥락에서 평가해야 하며, 평가 기준(criteria)을 먼저 정의하는 평가 주도 개발(evaluation-driven development)이 핵심이다.
- 도메인 역량, 생성 역량(사실 일관성·안전성), 지시 따르기 역량, 비용·지연 시간을 기준으로 모델을 선택하고, 공개 벤치마크와 리더보드의 한계를 이해해야 한다.
- 자체 모델 호스팅 vs API 사용을 7가지 축으로 비교하고, 신뢰할 수 있는 자체 평가 파이프라인(evaluation pipeline)을 설계하는 방법을 다룬다.

---

## 1. Evaluation Criteria (평가 기준)
> 애플리케이션을 만들기 전에 어떻게 평가할지 먼저 정의하는 평가 주도 개발이 AI 도입의 가장 큰 병목인 평가 문제를 해결하는 출발점이다.

평가가 쉬운 use case(추천, 사기 탐지, 코드 생성)가 우선 배포되지만, 평가하기 어렵다는 이유로 잠재력 큰 애플리케이션을 놓칠 수 있다. 평가 기준은 크게 도메인 특화 역량, 생성 역량, 지시 따르기 역량, 비용·지연 시간의 네 가지로 나뉜다.

### Domain-Specific Capability (도메인 특화 역량)
> 모델이 애플리케이션이 요구하는 특정 분야(코딩, 특정 언어 등) 역량을 갖췄는지 공개/비공개 벤치마크로 평가한다.

코딩은 functional correctness(기능적 정확성)로 평가하며, 효율성·메모리·가독성도 고려한다. 비코딩 역량은 주로 객관식 문제(MCQ, Multiple-Choice Question)로 평가하는데, MMLU 등이 대표적이다. 다만 MCQ는 좋은 답을 '구별'하는 능력을 측정할 뿐 '생성'하는 능력과는 다르며, 프롬프트의 작은 변화에 민감하다는 한계가 있다.

### Generation Capability (생성 역량)
> 개방형 출력의 품질을 평가하며, 과거 NLG(Natural Language Generation)에서 쓰던 유창성·일관성보다 사실 일관성과 안전성이 더 중요해졌다.

NLP(Natural Language Processing) 분야에서 발전해 온 fluency(유창성), coherence(일관성), faithfulness(충실성) 등이 기반이다. 모델 성능이 향상되면서 유창성 문제는 거의 사라지고, 환각(hallucination)과 안전성이 핵심 이슈로 부상했다.

#### Factual Consistency (사실 일관성)
> 출력이 주어진 context와 일치하는지(local), 또는 일반 지식과 일치하는지(global)를 검증한다.

가장 어려운 부분은 '무엇이 사실인가'를 정하는 것이다. AI as a judge가 가장 직접적인 방법이며, 더 정교한 기법으로 SelfCheckGPT(자기 검증), SAFE(검색 증강 검증)가 있다. 텍스트 함의(textual entailment) 문제로 framing하여 entailment/contradiction/neutral로 분류할 수도 있다. RAG(Retrieval-Augmented Generation) 시스템 평가의 핵심 기준이다.

```
[Response] --decompose--> [Statement 1]
                          [Statement 2]  --> Search API --> Verify
                          [Statement 3]
        (SAFE: Search-Augmented Factuality Evaluator)
```

#### Safety (안전성)
> 부적절한 언어, 유해한 조언, 혐오 발언, 폭력, 고정관념, 정치·종교 편향 등 출력이 해를 끼칠 수 있는 모든 영역을 평가한다.

범용 AI judge나 전용 toxicity 분류기(더 작고 빠르고 저렴함)를 사용하며, RealToxicityPrompts, BOLD 등의 벤치마크가 있다.

### Instruction-Following Capability (지시 따르기 역량)
> 모델이 지시를 얼마나 잘 따르는지를 측정하며, 아무리 좋은 지시도 모델이 못 따르면 무의미하다.

도메인 역량이나 생성 역량과 혼동되기 쉽다. IFEval은 자동 검증 가능한 형식(키워드, 길이, JSON 등) 준수를 평가하고, INFOBench는 내용·언어·스타일 제약까지 yes/no 기준으로 더 넓게 평가한다. 자신의 지시에 맞는 자체 벤치마크를 만드는 것이 권장된다.

#### Roleplaying (역할 연기)
> 가장 흔한 실세계 지시 유형 중 하나로, 모델에게 특정 캐릭터나 페르소나를 부여하는 것이다.

엔터테인먼트(게임, NPC)나 프롬프트 엔지니어링 기법으로 쓰인다. 평가 자동화가 어려워 RoleLLM, CharacterEval 등은 AI judge와 유사도 점수를 사용하며, 스타일과 지식(특히 '몰라야 하는' negative knowledge) 모두를 확인해야 한다.

### Cost and Latency (비용과 지연 시간)
> 품질이 좋아도 너무 느리거나 비싸면 쓸모없으므로, 품질·지연·비용을 균형 있게 최적화(Pareto optimization)해야 한다.

지연 시간 지표로 TTFT(Time To First Token), 토큰당 시간, 쿼리당 시간 등이 있다. API는 토큰 단위 과금이라 규모와 무관하게 단가가 일정하지만, 자체 호스팅은 규모가 커질수록 토큰당 비용이 저렴해진다.

---

## 2. Model Selection (모델 선택)
> 최고의 모델이 아니라 '내 애플리케이션에 최적인' 모델을 선택하는 것이 목표이며, 개발 과정에서 반복적으로 수행한다.

### Model Selection Workflow (모델 선택 워크플로우)
> 변경 불가능한 hard attribute(라이선스, 모델 크기, 프라이버시)와 개선 가능한 soft attribute(정확도, 독성)를 구분한다.

4단계 워크플로우: (1) hard attribute로 모델 필터링, (2) 공개 벤치마크·리더보드로 후보 좁히기, (3) 자체 평가 파이프라인으로 실험, (4) 프로덕션에서 지속 모니터링. 이 단계들은 반복적이다.

### Model Build Versus Buy (자체 구축 vs API 구매)
> 상용 API를 쓸지 오픈소스 모델을 자체 호스팅할지는 후보 모델 풀을 크게 줄이는 중요한 결정이다.

오픈소스 용어 구분: open weight(가중치만 공개), open model(학습 데이터까지 공개). 라이선스 검토 시 상업적 사용 허용 여부, 제약, 출력으로 다른 모델 학습 가능 여부를 확인해야 한다. 7가지 비교 축은 다음과 같다.

```
            API (Buy)              Self-Host (Build)
Data      | Send data out (risk) | Keep data internal
Perf      | Best closed models   | Slightly behind
Function  | Scaling, func-call   | logprobs access
Cost      | Per-token API cost   | Engineering effort
Control   | Rate limits, no freeze| Full control/freeze
On-device | Not possible         | Possible
```

데이터 프라이버시, 데이터 계보(copyright), 성능, 기능(function calling·logprobs), 비용(API vs 엔지니어링), 제어·접근·투명성, 온디바이스 배포의 7가지 축으로 결정한다.

### Navigate Public Benchmarks (공개 벤치마크 다루기)
> 수천 개의 벤치마크가 존재하며, 평가 하네스(evaluation harness)로 여러 벤치마크를 한 번에 실행할 수 있다.

공개 리더보드(Hugging Face Open LLM Leaderboard, HELM)는 소수 벤치마크를 집계해 모델을 순위 매긴다. 벤치마크 선택 기준이 불투명하고, 상관관계가 높은 벤치마크는 편향을 증폭시킨다. 집계 방식은 단순 평균 또는 mean win rate 등이 있다.

#### Data Contamination (데이터 오염)
> 모델이 평가 데이터로 학습되어 답을 외워 점수가 부풀려지는 현상으로, 매우 흔하다.

대부분 비의도적(인터넷 스크래핑)으로 발생한다. n-gram 중첩(정확하나 비쌈)이나 perplexity(저렴하나 부정확)로 탐지한다. 공개 벤치마크는 일부 데이터를 비공개로 유지하는 것이 바람직하다.

---

## 3. Design Your Evaluation Pipeline (평가 파이프라인 설계)
> 좋은 결과와 나쁜 결과를 구별하는 신뢰할 수 있는 평가 파이프라인이 AI 애플리케이션 성공의 관건이다.

### Step 1. Evaluate All Components (모든 구성요소 평가)
> 시스템의 각 구성요소와 end-to-end 출력을 독립적으로 평가하여 어디서 실패하는지 파악한다.

per task, per turn, per intermediate output 수준에서 평가한다. 사용자가 진짜 원하는 것은 작업 완수이므로 task-based 평가가 더 중요하지만, 작업 경계 설정이 어렵다.

### Step 2. Create an Evaluation Guideline (평가 가이드라인 작성)
> 평가에서 가장 어려운 부분은 출력이 좋은지 판단하는 것이 아니라 '좋다'가 무엇인지 정의하는 것이다.

애플리케이션이 해야 할 것과 하지 말아야 할 것을 모두 정의한다. 정확한 응답이 항상 좋은 응답은 아니다. 각 기준마다 점수 체계(binary, 1~5 등)와 예시가 포함된 rubric을 만들고, 평가 지표를 비즈니스 지표(예: 사실 일관성 80% → 고객지원 30% 자동화)에 연결한다.

### Step 3. Define Evaluation Methods and Data (평가 방법과 데이터 정의)
> 기준마다 다른 평가 방법(toxicity 분류기, 의미 유사도, AI judge)을 혼합하고, 주석된 평가 데이터를 준비한다.

logprobs가 있으면 모델 확신도 측정에 활용한다. 자동 지표를 최대한 쓰되 프로덕션에서도 인간 평가를 병행한다. 데이터를 slice(세분화)하여 편향을 피하고 디버깅하며, Simpson's paradox(부분집합마다 우수하나 전체에서 열등)를 주의한다. 여러 평가 세트를 두고, 부트스트랩으로 평가 세트 크기의 신뢰성을 검증한다.

```
Difference to detect | Sample size (95% confidence)
        30%          |        ~10
        10%          |        ~100
         3%          |        ~1,000
         1%          |        ~10,000
  (Rule: 3x smaller diff -> 10x more samples)
```

평가 파이프라인 자체도 평가해야 한다: 올바른 신호를 주는가, 재현 가능한가(AI judge는 temperature=0), 지표 간 상관관계, 추가되는 비용·지연. 니즈가 변하면 평가 기준도 진화하므로 실험 추적(experiment tracking)을 하며 반복(iterate)한다.

---

## Summary (핵심 정리)
- 신뢰할 수 있는 평가 파이프라인의 부재가 AI 도입의 가장 큰 장벽이며, 도메인·생성·지시 따르기·비용 기준으로 모델을 평가해야 한다.
- 모델 자체 호스팅 vs API 결정은 데이터 프라이버시·성능·비용 등 7가지 축으로 팀마다 다르게 판단한다.
- 공개 벤치마크는 나쁜 모델을 걸러낼 뿐 최적 모델을 찾아주지 못하며(오염 위험), 명확한 가이드라인과 rubric 기반의 자체 평가 파이프라인을 설계해야 한다.
