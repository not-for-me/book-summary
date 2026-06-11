# 10. Composable Agentic Workflows

## 챕터 개요 (3줄 요약)
- 앞 9개 chapter의 패턴들을 엮어, 시간이 지날수록 개선되는 production-ready agentic application을 구축하는 방법을 vertical slice로 시연한다.
- 교육 콘텐츠 생성 workflow(Pattern 23 use case)를 multiagent 프레임워크 없이 simple·composable 패턴(Anthropic 권장, Unix 철학)으로 LLM·cloud agnostic하게 구현.
- copilot(AI 보조)·agent(자율) 모드를 지원하며, 5개 아키텍처 컴포넌트와 지속 학습으로 자율성을 키운다.

---

## 1. Agentic Workflow & 실행
> AI 보조(copilot)~자율(agent) 스펙트럼의 application을 agentic이라 하며, 교육 콘텐츠 생성 workflow의 vertical slice를 시연한다.

- Pattern 23과 달리 review 2단계, 데이터 전송 명시, GenAI writer(이 책 내용 기반) 추가, 프레임워크 미사용.
- **setup**: venv + `pip install`, Gemini API key. `utils/llms.py`에서 BEST/DEFAULT/SMALL 3개 모델로 quality·cost·speed trade-off. logging은 prompts.log·guards.log·feedback.log.
- **copilot 모드**: `streamlit run` — 사용자가 topic 입력·Next 클릭으로 workflow 진행, 초안 직접/chat 편집(artifact).
- **agent 모드**: `cmdline_app.py` — 모든 AI 추천 수락+변경 없음과 동일한 자율 실행.
- **workflow**: Task Assigner가 topic에 맞는 writer 선택(기본 history) → 사용자 변경 시 human feedback 로깅.

## 2. System Architecture (5개 컴포넌트)
> agent·multiagent orchestration·governance·learning pipeline·data program이 상호작용한다.

- **agent patterns**: 각 step을 독립 agent로 구현 — CoT(13), RAG(6)·Index-Aware(9), Tool Calling(21), Reflection(18)·Self-Check(31), Template(29)·Assembled Reformat(30). PanelSecretary는 PydanticAI(LLM/cloud agnostic), system prompt를 Jinja2 템플릿 파일에서 read, DEFAULT_MODEL, retries=2(try-and-try-again은 성공률 90%+면 허용). GenAI writer는 LlamaIndex(semantic RAG). 사용자 chat 수정은 Long-Term Memory(28)로 저장·재적용.
- **multiagent architecture**: agent mode는 순차 호출(`write_about`: find_writer→draft→panel review→revise). copilot은 각 페이지가 "자기" agent 호출, Grammar(2)로 structured output, Prompt Caching(25, `@st.cache_resource`)로 재호출 방지. 분기·로직은 직접 구현(프레임워크 미사용의 이점).
- **governance/monitoring/security**: 입력 Guardrails(32)를 LLM-as-Judge(17)로 구현(InputGuardrail에 condition 전달), asyncio.gather로 guardrail을 원 작업과 병렬(실패 시 둘 다 종료). guards.log 로깅, Degradation Testing(27)으로 병목 점검. access control·audit·human-in-the-loop도 구현.
- **learning pipeline**: copilot이 다음 agent로 넘기기 전 사용자 편집 여부 확인 → human feedback(record_human_feedback) 로깅. prompts.log·evals.log로 offline 평가·post-training. 결과(appeal·engagement·exam 성적)는 며칠 후 알 수 있어 step-outcome 연결 필요. feedback은 Content Optimization(5)·Adapter Tuning(15)·Prompt Optimization(20)에 활용(대량 시 sampling).
- **data program**: organic feedback만으론 부족(data size·complexity·detailed feedback·Automation Paradox·incorrect label). Evol-Instruct(16)로 복잡 변형 생성, Self-Check(31)로 문제 지점 표시.

## 3. Deployment
> 각 agent가 독립 배포 가능한 composable 구조로 monolithic 대비 이점이 많다.

- **이점**: modularity·reusability(Dependency Injection 19로 독립 개발·테스트), technical flexibility, 표준 protocol·tool(PydanticAI·LlamaIndex·Mem0·MCP·A2A), independent scaling, failure isolation, 빠른 개발, security·compliance 재활용.
- open source Python 기반, serverless 배포 가능. frontend TypeScript + backend Python 혼합도 흔함.

---

## Summary (핵심 정리)
- 책 전체 design pattern을 통합해 production-ready agentic application을 구축 — 교육 콘텐츠 생성을 simple·composable 패턴 위에.
- copilot(보조)·agent(자율) 두 모드, 지속 학습으로 copilot을 점점 자율적으로 발전.
- 5개 아키텍처 컴포넌트: 개별 agent, multiagent orchestration, 입력 guardrail, human feedback 학습 pipeline, 데이터 creation·collection·curation 프로그램.
- composable 접근(프레임워크 의존 최소)이 modularity·scaling·failure isolation·빠른 개발을 제공.
- GenAI를 실용적으로 만드는 것은 이 패턴들의 조합 — 위험·비용·신뢰성·확장성을 함께 다룬다.