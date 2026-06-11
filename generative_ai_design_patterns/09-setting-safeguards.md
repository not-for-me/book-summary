# 09. Setting Safeguards

## 챕터 개요 (3줄 요약)
- foundational model은 nondeterministic·general-purpose라 부정확·hallucination·misalignment 위험이 항상 존재 — 안전장치를 두는 4개 패턴(29~32).
- Template Generation(사전 검토 템플릿), Assembled Reformat(저위험 2단계 분리), Self-Check(token 확률로 hallucination 탐지), Guardrails(포괄적 보호 레이어).
- 위험과 비용·확장성·복잡성 사이의 균형을 맞춘다.

---

## 1. Pattern 29: Template Generation
> human review가 필요한 항목 수를 줄이기 위해 사전에 템플릿을 생성·offline 검토하고, inference 시 deterministic string 치환만 수행한다.

- **문제**: 감사 편지처럼 위험이 높지만 수천 건/일이라 human review가 확장 불가.
- **해법**: LLM이 thank-you note 자체가 아닌 **템플릿**을 생성 → human 검토·편집 → inference 시 placeholder(`[CUSTOMER_NAME]`) 치환만. deterministic·constrained이라 오류·toxic 위험 최소.
- **예시**: 패키지 투어 — destination×package×language(3×4×2=24) 조합별 템플릿 사전 생성·DB 저장, 예약 시 retrieve 후 치환.
- **고려사항**: 조합 수가 tractable할 때 유효(너무 많으면 Assembled Reformat, 또는 Guardrails). ML과 결합 가능(personalized landing page + propensity model로 next best action). mail merge의 LLM 버전.

## 2. Pattern 30: Assembled Reformat
> 콘텐츠 생성을 2개 저위험 단계로 분리 — ① 저-hallucination 방법(OCR·RAG·Tool Calling·Template)으로 raw data 조립, ② LLM으로 재서술·요약.

- **문제**: 제품 카탈로그 수십만 페이지 — 카메라 배터리를 lithium→alkaline로 hallucinate하면 항공 화재 등 고위험. 동적 생성은 너무 위험.
- **해법**: hallucinate되면 안 될 특성을 DB/문서 추출로 조립(정확하나 읽기 어려움) → LLM에 context로 넣어 rephrase/reformat(rewording·summarizing은 부정확 도입 가능성 낮음). 결과는 유창하고 목적에 맞음.
- **예시**: paper machine 부품 — part_name·id·warranty·price는 DB, description·failure_modes는 manual(temperature 0.1)에서 추출 → CatalogContent → SEO·설득력 있는 Markdown으로 재포맷(추출된 3개 failure mode에 grounding).
- **고려사항**: 두 단계 모두 저위험이나 검증 필요(2가지 방법으로 추출 비교, Self-Check, LLM-as-Judge로 raw data 유지 확인). Template Generation이 가능하면 그쪽 우선(전체 검토 가능). 동적 콘텐츠(마케팅 landing)엔 Template Generation 필요.

## 3. Pattern 31: Self-Check
> token 확률(logprob)로 LLM 응답의 hallucination을 탐지 — factual query의 저신뢰 답변에 대한 안전장치.

- **문제**: hallucination(통계적 token 생성). 이미지에서 숫자 추출은 90~97% 정확(3~10% hallucinate), chain이 길수록 오류 compound. 비중첩 LLM 3개 비교는 비용·데이터 중첩으로 비현실적.
- **해법**: logprob에서 `e^logit`으로 token 확률 계산. 압도적 선호 token은 ~100%, 여러 후보면 낮음. OpenAI `logprobs=True, top_logprobs=5`.
- **logprob 거동**: Ataturk 출생연도 1881은 confident(저확률 시작 토큰은 "Mustafa Kemal Atatürk" 같은 valid 대안 때문 — false positive 주의). hallucination(John Cole Howard)은 The·Ed에서 저확률.
- **false positive 제한**: ① tokens of interest 식별(structured output의 핵심 값만), ② sample sequence(여러 생성이 답에 동의하는지, embedding 비교), ③ perplexity(길이 정규화), ④ ML 모델(가장 robust, 모든 feature 결합).
- **예시**: 영수증 4숫자 추출 — paid_amount가 checksum. imputation 없으면 confidence>0.9, 1개면 ~0.55, 2개면 ~0.17로 문제 행 식별.
- **고려사항**: 더 간단한 대안 — 모델에 "I don't know" 출구 제공(`float | Literal["Unknown"]`). RAG의 모순 chunk 식별에 유용. 일부 모델은 logprob 미제공.

## 4. Pattern 32: Guardrails
> LLM의 입력·출력·context·tool 파라미터에 작동하는 코드 레이어로 악의적 행위자로부터 보호하고 허용 범위 내 작동을 보장한다.

- **문제**: security(prompt injection·jailbreak, 직접/간접), data privacy(PII·trade secret 노출), content moderation(toxic·harmful), hallucination, alignment(정책·brand voice·편향). 앱 코드에 흩뿌리면 유지보수 어려움.
- **해법**: 입력 preprocessing/출력 postprocessing 레이어(수정 또는 거부).
- **prebuilt**: Gemini safety_settings, NVIDIA NeMo·Guardrails AI·LLM Guard. LLM Guard Toxicity·PromptInjection(post-trained SLM 사용)·Regex(redact).
- **custom**: 코드 로직, foundational model prompting(LLM-as-Judge로 banned topic 거부), SLM post-training. 동일 signature로 순차 적용.
- **예시**: Jane Austen RAG — PII 이름 치환(Mr. Darcy→a man), banned topic(religion·politics) 거부. `GuardedQueryEngine`이 입력·출력에 guardrail 적용.
- **고려사항**: engineering 복잡성·latency 증가 → 정당한지 확인. SLM으로 latency 관리, 입력/retrieval guardrail을 요청과 병렬 실행(asyncio.gather). security-usability-performance trade-off, 공격은 진화하는 arms race → guardrail을 몇 달마다 교체 가능한 model-agnostic wrapper로 설계, evaluation dataset 갱신.

---

## Summary (핵심 정리)
- foundational model의 위험(security·privacy·moderation·hallucination·alignment)에 4개 안전장치 패턴.
- **Template Generation**: 사전 검토 템플릿 + deterministic 치환, human review 확장 불가 시.
- **Assembled Reformat**: 저위험 조립 + LLM 재포맷, 정확한 콘텐츠를 매력적으로 표현.
- **Self-Check**: logprob token 확률로 hallucination 탐지(tokens of interest·sampling·perplexity·ML로 false positive 제한).
- **Guardrails**: 입출력·context·tool에 보호 레이어(prebuilt/custom), 악의적 공격 대비.
- 위험 vs 비용·확장성·복잡성 균형 — 가능하면 더 단순한 패턴(Template > Assembled Reformat, "I don't know" 출구) 우선.