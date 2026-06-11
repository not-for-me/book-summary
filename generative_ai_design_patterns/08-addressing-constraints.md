# 08. Addressing Constraints

## 챕터 개요 (3줄 요약)
- LLM을 production에 배포할 때 마주하는 computational overhead·latency·cost·memory 제약을 다루는 5개 패턴(24~28).
- SLM(distillation·quantization·speculative decoding), Prompt Caching, Inference Optimization, Degradation Testing, Long-Term Memory.
- PoC에서 작동하는 모델과 수천 사용자를 서빙하는 시스템 사이의 간극을 메우는 toolkit.

---

## 1. Pattern 24: Small Language Model (SLM)
> 품질을 크게 희생하지 않고 cost·latency 제약에 맞는 작은 모델을 쓰는 기법 — distillation, quantization, speculative decoding.

- **문제**: frontier LLM은 고사양 GPU·메모리 필요(Llama 4 Scout = 4×H100, >$10/h), 파라미터 많을수록 느림. 단순히 작은 모델로 바꾸면 복잡 task 품질 저하.
- **distillation**: teacher(큰 모델) 응답을 student(작은 모델)가 mimic하도록 fine-tune. KL divergence(student log_softmax, teacher softmax), temperature scaling으로 "dark knowledge" 보존, `loss=(1-α)·task_loss + α·distillation_loss`. narrow-scope 응용에 적합, meta/ensemble distillation 가능.
- **quantization**: FP32→INT8/4로 정밀도↓ 메모리↓(지식은 거의 유지). pretraining(QAT), training 중(mixed-precision·dynamic), post-training(weight-only GPTQ/AWQ, full-model, QLoRA·SPQR·BitNet). `BitsAndBytesConfig(load_in_4bit, nf4, double_quant)`.
- **speculative decoding**: 작은 draft 모델이 토큰 제안 → 큰 target 모델이 병렬 검증, 동의 시 수락·불일치 시 재생성. 정확도 유지하며 latency↓ (쉬운 토큰은 작은 모델로). vLLM `speculative_model`.
- **예시**: Python 문서화 SLM — Gemma 3 12B→1B distill 후 4-bit quantize(수분→19초).
- **한계/대안**: distillation은 generality 상실·teacher bias 계승, quantization은 정밀도-효율 trade-off·하드웨어 지원 의존. 대안: model sharding, parallelization, continuous batching, Prompt Caching, QAT 모델, Adapter Tuning.

## 2. Pattern 25: Prompt Caching
> 동일/유사 prompt의 응답(client-side) 또는 모델 내부 상태(server-side)를 재사용해 주로 cost, 때로 latency를 줄인다.

- **문제**: 반복 요청(케이블사 31% 장애 문의, 은행 30% 로그인) 재계산은 hardware·user time·cost 낭비.
- **client-side(memoization)**: KV response cache(key=prompt, value=response). 정확 일치 필요 → network call·LLM 호출 없어 latency·cost↓. LangChain InMemoryCache/Redis, OpenAI OPENAI_CACHE_DIR.
- **semantic caching**: canonical form 키, 다중 유사 키 저장, embedding 유사도 검색(GPTCache). 단점: 유사 query에 동일 응답으로 nuance 상실.
- **server-side(prefix caching)**: 공통 prefix(system prompt·예시)의 내부 상태 재사용 → 창의성 유지, **TTFT 크게 감소**(streaming에 유익). provider 암묵적 캐싱(>1024 token), vLLM도 지원. context caching(Gemini 멀티미디어).
- **고려사항**: multitenant 정보 유출 위험(user ID를 cache key에), 캐시 invalidation(TTL, 모델 버전 변경 시 전체 무효화). client-side는 전체 단축(latency↓), server-side는 TTFT만.

## 3. Pattern 26: Inference Optimization
> self-host LLM의 inference 효율을 continuous batching·speculative decoding·prompt compression으로 개선.

- **문제**: 규제 데이터로 self-host 시 GPU 제약·실시간 응답 기대.
- **continuous batching**: prompt 길이가 제각각이라 전통 batching(padding) 비효율 → 큐에서 요청을 GPU 코어 free 시 즉시 slot, forward pass마다 stop token 체크 후 교체. vLLM/SGLang 기본 제공(개별 아닌 전체 요청 제출). 20x throughput.
- **speculative decoding**: draft(작은)·target(큰) 모델 협업, draft가 토큰 제안·target 검증·정정. ~14% 빠름(num_speculative_tokens 튜닝 중요).
- **prompt compression**: ① hard(redundant 제거·약어·키워드, 사람 읽기 가능, 정보 손실 체크), ② soft(encoder로 연속 vector `<bach_1>...`로 인코딩, model-specific, 500xCompressor). KV cache 메모리↓.

## 4. Pattern 27: Degradation Testing
> AI 애플리케이션의 성능 병목을 식별 — 실패 지점이 아닌 품질이 degrade하기 시작하는 지점과 제약을 파악한다.

- **문제**: 전통 load testing(400/500 에러)은 부족 — 5% 요청이 느려지기 시작하는 degradation point와 원인 제약 파악 필요.
- **core metrics**: ① TTFT(첫 토큰까지, attention·입력 길이·KV cache 좌우 → Prompt Compression·Caching·context window 축소), ② EERL(전체 응답, 큐·network 포함 → 출력 토큰 축소·parallelization·speculative execution), ③ TPS(전체 throughput, saturation point), ④ RPS(완료 요청/초).
- **scalability/resilience**: scalability testing(점진 부하 증가, breaking point), stress analysis(한계 초과, recovery time), load testing(예상 peak 검증).
- **대응**: 메모리 full이면 GPU 업그레이드, multi-GPU 분산(data/model parallelism), 패턴 적용(TTFT엔 distillation·quantization·continuous batching, throughput엔 distillation·quantization·speculative decoding).
- **도구**: LLMPerf, LangSmith, Arize Phoenix, vLLM/SGLang 벤치마크, PagedAttention(vLLM).

## 5. Pattern 28: Long-Term Memory
> 모델이 사용자 상호작용 전반에 걸쳐 정보를 유지 — working·episodic·procedural·semantic memory.

- **문제**: LLM은 각 prompt를 stateless·독립 처리. 전체 이력 prepend는 context window·비용(quadratic scaling) 한계.
- **working memory**: 현재 세션 메시지 유지("change it to large"의 it). token limit로 trim(유효한 message 경계, system prompt 유지). LangChain `trim_messages`.
- **episodic memory**: 이전 세션 메시지를 persistent store에 저장·검색(recency 기반, 메타데이터 필터). 거부했던 제안 반복 회피.
- **procedural memory**: system instruction·user profile(알레르기 등 추출). 응답 personalize.
- **semantic memory**: 내용 기반 fact 저장·검색(recency보다 content). LangGraph PostgresStore.
- **예시(Mem0)**: vector store·embedder·LLM·DB 구성. `memory.add(conversation, user_id)`로 LLM이 personal fact 추출·embed·저장, `memory.search(query, user_id)`로 유사 메모리 검색·ranking. user_id로 세션 간 지속, run_id로 단기, filter로 카테고리 제한.
- **고려사항**: production은 microservice로 배포, background thread로 기록(UX 저하 방지). 일반적으로 episodic보다 semantic memory 선호(메시지 원본 저장은 retrieval latency·cache miss 디버깅 어려움).

---

## Summary (핵심 정리)
- production LLM의 overhead·latency·cost·memory 제약을 5개 패턴으로 해결.
- **SLM**: distillation(지식 범위 축소)·quantization(정밀도↓)·speculative decoding(작은+큰 모델)로 비용·지연 절감.
- **Prompt Caching**: 반복 응답 재사용 — client-side(전체 단축)·server-side prefix(TTFT↓)·semantic.
- **Inference Optimization**: continuous batching·speculative decoding·prompt compression으로 self-host 효율↑.
- **Degradation Testing**: TTFT·EERL·TPS·RPS + scalability/resilience로 품질 저하 지점·제약 식별.
- **Long-Term Memory**: working/episodic/procedural/semantic 4종 memory(Mem0)로 context window 한계 없이 이력 유지.