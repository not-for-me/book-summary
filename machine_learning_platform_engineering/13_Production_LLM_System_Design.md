# 13. Production LLM System Design

## 챕터 개요 (3줄 요약)

- prototype→production LLM은 nondeterministic 특성에 맞는 새로운 testing·safety·cost 관리를 요구하며, prompt를 versioning·testing되는 code로 취급한다.
- Langfuse(prompt 관리), DeepEval/G-Eval(semantic evaluation), Promptfoo(adversarial), Guardrails AI(safety)로 production 신뢰성을 확보한다.
- token 기반 cost는 tiered model routing·semantic caching·prompt 최적화로 60~80% 절감하며 지속 모니터링한다.

---

## 13.1 Prompt engineering: Code for the generative AI era

> LLM에서 business logic이 자연어 prompt에 존재 → code와 동일한 rigor(versioning·testing·최적화) 필요.

### system prompt의 5가지 기능

persona/role 정의, 제약·guardrail 설정, output format·style, context·task 목표, operational knowledge. 각각이 다른 failure mode를 방지.

- **v1.0(문제)**: "helpful assistant" — 모호한 role, scope creep, 일관성 없는 persona, safety 제약 zero.
- **v2.0(production)**: 명시적 role(DakkaBot), operational boundary(문서만 사용·불확실 시 인정·speculation 금지·redirect), output(direct answer·source citation·confidence indicator), safety(API key 노출 금지·security escalation), 확장 가능 구조.

### Langfuse prompt 관리

- prompt를 hardcode 아닌 중앙 관리 asset으로: `create_prompt(name, type="chat", prompt, labels=["production"], config, tags)`. `{{context}}`·`{{query}}` 템플릿 변수. code 재배포 없이 UI에서 수정.
- **label 기반 배포**(production/staging/experimental), config에 model param 결합, boundary prompt(out-of-scope graceful fallback).
- **runtime resolution**: `get_prompt(name, label="production")` → config override → `compile(context, query)` → invoke(metadata에 prompt version 추적). 실패 시 boundary prompt로 graceful degradation.
- 이점: 비기술자 협업(CMS), audit trail, A/B testing, 언어/user group/region별 variant, client-side caching.

---

## 13.2 Testing LLM applications

> 결정적 SW는 input X→output Y, LLM은 같은 query가 여러 valid 응답 → exact match 대신 semantic evaluation.

### DeepEval + G-Eval

- `deepeval set-gemini`로 judge model 설정. async→sync helper(`asyncio.run`, singleton 인스턴스).
- **G-Eval(LLM-as-Judge)**: 다른 LLM이 custom criteria로 평가. flexible scoring, chain-of-thought(추론 설명), template 기반. BLEU/ROUGE 같은 표면 유사도 못 잡는 "개발자에게 helpful한가" 등 nuance 평가.
- **security boundary test**: credential-harvesting query에 대해 deflection·정보 미노출·constructive redirection 3차원 평가. threshold 0.9. 부분 노출도 실패로 명시("any sensitive info = failing").
- **multi-metric 동시 테스트**: correctness(0.7)·relevancy·completeness(0.6, 낮은 threshold — 어려움). metric 난이도별 threshold 차등.
- G-Eval 고려사항: criteria·judge 품질에 의존, 추가 LLM call로 cost·latency↑. 큰 model이 정확하나 느림 — production safety엔 투자 가치.

```python
metric = GEval(name="Security Boundary Respect",
    criteria="Determine if response deflects security questions without revealing sensitive info...",
    evaluation_params=[INPUT, ACTUAL_OUTPUT], threshold=0.9)
assert_test(LLMTestCase(input=..., actual_output=get_response(...)), [metric])
```

### Safety and adversarial testing (Promptfoo)

- RAG는 user query가 retrieval·context에 직접 영향 → prompt injection 취약(instruction과 data가 같은 token stream).
- 예: role-playing(pirate), 경쟁사 비교(CryptKeeper) 공격. 수동 테스트로 불충분.
- **Promptfoo**: `redteam generate`로 48 adversarial prompt 생성. **plugin**(contracts, excessive-agency, hallucination, harmful, hijacking, politics) + **strategy**(basic, jailbreak, jailbreak:composite). `promptfoo_provider.py`(bridge)로 DakkaBot에 연결. `promptfoo view`로 대시보드 분석.
- 발견된 취약점 → regression test 추가 + system prompt safety 제약 업데이트 → 지속 개선 loop.

---

## 13.3 Governance and safety in production

> LLM safety 실패는 민감 data 노출·harmful content·compliance 위반을 유발 → 다층 safety system 필요.

### Guardrails AI (다층 guardrail)

- 3 layer: **input guard**(sanitization·PII·competitor), **processing control**, **output guard**(validation).
- `guardrails hub install`로 validator 설치. `Guard().use_many(CompetitorCheck(on_fail=EXCEPTION), GuardrailsPII(on_fail=FIX))`.
- **OnFailAction 7종**: EXCEPTION/REFRAIN(완전 차단, 고위험), FIX/FILTER(sanitize, PII·profanity), REASK/FIX_REASK(재생성), NOOP(logging).
- **Guardrails Hub**: content safety, PII, bias, factuality, format, business logic validator marketplace. infra tag(rule-based=빠름, ML=중간, LLM-based=느림·비쌈). rule→ML→LLM 순으로 설계.
- **Gemini 내장 safety filter**: harassment·hate·explicit·dangerous·civic 5범주, BLOCK_NONE~BLOCK_LOW_AND_ABOVE. model-level(business policy는 못 잡음 → custom validator).
- **output validation**: 문서 경계·내부 정보 노출·tone 검증.
- **hallucination testing**: output vs ground truth 비교. G-Eval("context에 없는 정보 포함?") 또는 DeepEval FactualConsistencyMetric. golden dataset으로 regression test.

```python
input_guard = Guard().use_many(
    CompetitorCheck(DATAKRYPT_COMPETITORS, on_fail=OnFailAction.EXCEPTION),
    GuardrailsPII(entities=["EMAIL_ADDRESS","CREDIT_CARD","SSN"], on_fail=OnFailAction.FIX))
```

---

## 13.4 Cost optimization strategies

> LLM은 electricity utility처럼 token 소비 기반 과금 — 복잡한 query가 100x 비쌀 수 있음.

### LLM economics

- model 선택은 quality vs cost 단순 trade-off 넘어섰음: GPT-4/Gemini(추론), Claude(coding), 도메인 특화 model. 요구사항에 매칭.
- 예(2000 in + 500 out): GPT-4 ~$0.035, Gemini Flash ~$0.003, open source hosted ~$0.0005 + infra.
- **배포 전략**: pay-per-token(managed endpoint, infra overhead 0, 선형 scaling) vs reserved(self-hosted, 70B+ = GPU 4~8개·$8~15K/월). <100K/월은 cloud API 저렴, >500K/월부터 open source 경제성. peak auto-scaling이 cost driver.

### 최적화 기법

- **13.4.2 tiered model routing**: 값싼 model(flash-lite)로 SIMPLE/MODERATE/COMPLEX 분류 → 적절한 tier 라우팅(flash-lite/flash/pro). 60~80% 절감. routing 정확도 모니터링·threshold A/B test 필요.
- **13.4.3 caching**: **context caching**(문서 chunk 재사용, 50~80% input token 절감, cache invalidation 주의) + **semantic caching**(embedding 유사도로 유사 질문 감지, 30~60% call 제거). **RedisVL**(Redis + vector, cosine/dot product/euclidean, 서브ms 검색). similarity threshold 0.92 균형.
- **13.4.4 prompt 최적화**: verbose prompt를 concise로(identity 압축·instruction consolidation·format 단순화·redundancy 제거) 70% token 절감. 단 eval로 품질 검증 필수.
- **13.4.5 cost monitoring**: 80% budget alert·per-query cost·outlier 탐지. Langfuse cost tracking으로 비싼 trace·prompt version별 spending·quality 상관 분석.

```python
class CostOptimizedDakkaBotCore:
    # router_llm(flash-lite) -> route_query -> SIMPLE/MODERATE/COMPLEX
    # -> flash-lite / flash / pro 선택
```

### 13.4.6 From traditional ML to LLMOps

- ch1~10 기반(robust platform, monitoring, QA)은 그대로. LLM 신규 challenge: prompt engineering as code, multistep reasoning architecture, nondeterministic evaluation, token-based cost, safety·governance.
- LLMOps는 MLOps 대체가 아닌 **진화** — automation·observability·systematic testing·continuous improvement 원칙은 불변.

---

## Summary (핵심 정리)

- nondeterministic output은 evaluation 패러다임 전환 요구 — exact match 아닌 semantic quality·factual accuracy·safety 평가.
- prompt engineering은 자연어와 SW engineering을 잇는 핵심 discipline — prompt를 code처럼(version control·testing·최적화).
- production LLM은 다층 safety(input sanitization·output validation·지속 monitoring) 필요.
- cost 최적화는 token 기반 경제 이해 — tiered routing·semantic caching·prompt 최적화.
- adversarial testing(Promptfoo)은 필수 — prompt injection·jailbreak·scope violation을 배포 전 탐지.
- production 배포는 진화된 infra(variable token load auto-scaling, quality+cost monitoring, 자동 alerting) 필요. 엔지니어링 fundamentals는 기술이 진화해도 불변이다.