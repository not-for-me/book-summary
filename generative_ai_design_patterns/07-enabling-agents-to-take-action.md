# 07. Enabling Agents to Take Action

## 챕터 개요 (3줄 요약)
- 콘텐츠 생성을 넘어 application이 세상과 상호작용(정보 획득·환경 변경)하게 하는 3개 패턴 — 이 선을 넘으면 agentic으로 본다.
- Tool Calling(외부 함수 호출), Code Execution(코드 생성·실행), Multiagent Collaboration(전문 agent 조직화).
- Tool Calling + Reflection이 agentic application의 임계 행동; MCP·A2A 프로토콜로 상호운용성 발전.

---

## 1. Pattern 21: Tool Calling
> LLM이 함수 호출이 필요하다고 판단하면 special token과 인자를 emit하고, client-side postprocessor가 함수를 실행해 결과를 LLM에 돌려준다 (= Grammar의 확장).

- **문제**: LLM은 콘텐츠 생성만 가능 — 계산·항공권 예약·환불 같은 실제 동작 불가("환불됐다"는 텍스트만 생성).
- **동작**: LLM이 `[CALL_TOOL: book_flight, ...]` emit → 프로그램이 파싱·타입변환·함수 호출 → 반환값을 응답에 삽입. **보안상 LLM이 직접 호출하지 않음** — client가 실행.
- **여는 use case**: 최신 지식(RAG보다 동적), personalization, enterprise API, 계산, ReAct(reasoning+action 교차 = CoT+Tool Calling).
- **OpenAI 5단계**: 함수 구현 → tool 정의 전달 → client에서 function_call 처리·실행 → 결과 append 후 재호출 → 최종 응답. self-descriptive 함수명·docstring 중요.
- **LLM별 차이**: OpenAI(parameters), Anthropic(input_schema), Llama(function 키 하위) → LLM-agnostic 프레임워크(PydanticAI/LangChain/LangGraph/LiteLLM) 사용 권장.
- **MCP**: `@mcp.tool()` annotation으로 표준화. stdio(동일 언어) 또는 streamable-http(원격). MCP client가 server stub, LangGraph로 ReAct agent 생성.
- **예시**: 날씨 질문 — latlon_geocoder(Google Maps) + get_weather_from_nws 두 tool을 MCP server로 노출, ReAct agent가 자동 호출.
- **고려사항**: tool 3~10개로 제한이 정확도 높음, 결정적 정보는 모델에 안 맡김, 오류 시 descriptive 메시지(Reflection 활용). MCP 한계: security(인증 미강제), collaboration(단방향, A2A/ACP 보완), streaming(30~60초 종료).
- **prompt injection**: tool 호출 시 공격 피해 확대 → 6가지 방어(Action-Selector, Plan-Then-Execute, Map-Reduce, Dual-LLM, Code-Then-Execute, Context-Minimization).

```
user ─▶ LLM ─(emit tool call + args)─▶ client postprocessor ─▶ external API
                ▲                                                    │
                └──────────── result inserted into response ◀────────┘
```

## 2. Pattern 22: Code Execution
> LLM이 코드/DSL을 생성하고 외부 system(보통 sandbox)이 실행해 최종 결과를 만든다.

- **문제**: graph 생성·이미지 annotation은 Tool Calling 부적합(짧은 인자가 아닌 DSL/긴 구문 입력). DB 함수가 SQL을 받는 경우도 마찬가지.
- **해법**: LLM이 Matplotlib/SQL/Mermaid/DOT 등 DSL 생성 → sandbox에서 실행. ReAct의 일부 action으로도 가능.
- **예시**: 농구 토너먼트 결과 그래프 — LLM이 Graphviz DOT 생성 → `dot -Tpng`로 이미지 렌더링.
- **고려사항**: 반드시 sandbox(CPU·memory·network·시간 제약, Docker/VM)에서 실행. 실행 전 검증(syntax·static analysis) 권장. compiler 오류를 LLM에 되돌려 Reflection으로 재시도. 좁은 DSL + parser 환경에서 가장 안정적. DB 업데이트는 SQL 한 transaction이 무결성 유지에 유리.

## 3. Pattern 23: Multiagent Collaboration
> 전문화된 single-purpose agent들을 인간 조직 구조처럼 조직화해 단일 LLM 호출의 한계를 넘는다.

- **단일 agent 한계**: cognitive bottleneck(유한 context), parameter 효율 저하, reasoning depth 제한(순차적), domain adaptation 문제(catastrophic forgetting).
- **다중 전문 agent 이점**: task decomposition, parallel processing, hierarchical 문제 해결, domain/functional 전문화 → 수평 확장·robustness·emergent capability.
- **아키텍처**: ① hierarchical(executive-worker, prompt chaining = sequential workflow, router), ② peer-to-peer(voting/consensus, CrewAI), ③ market-based(auction·utility, sealed-bid/English auction), ④ human-in-the-loop(human-proxy agent가 충돌 해결).
- **use case**: breadth-first/parallel(가장 흔함), 복잡 reasoning, multistep, 협업 콘텐츠 생성, adversarial verification(red/blue team), domain 통합, self-improving.
- **예시(AG2)**: 9학년 워크북 — human→Task Assigner(router)→writer(history/math) 초안→review panel(round-robin, district admin·parent 등 다관점)→secretary 요약→writer 재작성→최종.
- **고려사항**: Anthropic은 복잡 프레임워크보다 simple·composable 패턴 권장. peer로 병렬화해 wall-clock 단축. interagent 통신 overhead 비선형 증가, consistency 유지 어려움. **A2A 프로토콜**로 다른 프레임워크·머신 agent 상호 통신. 2025 분석: 다중 agent task의 40~80%가 실패(14 failure mode — specification/interagent misalignment/task verification). 단일 agent로 충분하면 그쪽을 권장.

---

## Summary (핵심 정리)
- 3개 패턴이 모델을 passive 처리기에서 active 참여자로 전환 — 외부 tool 접근·코드 실행·팀 협업.
- **Tool Calling**: special token으로 함수 호출 emit, client가 실행(LLM은 직접 호출 안 함), MCP로 표준화, prompt injection 방어 필요.
- **Code Execution**: DSL 생성 → sandbox 실행, graph·DB 등 Tool Calling 부적합 task용.
- **Multiagent Collaboration**: 전문 agent를 hierarchical/peer-to-peer/market 구조로 조직화, A2A로 상호운용, 단 실패율·overhead 주의.
- Tool Calling + Reflection = agentic application의 임계 행동.