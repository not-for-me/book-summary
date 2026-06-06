# 15. Building a QA Agent with LangGraph

## 챕터 개요 (3줄 요약)

- 14장의 전문가 에뮬레이션 개념을 LangGraph(오케스트레이션)와 Streamlit(프론트엔드)으로 통합해 실제 KG 질의응답 애플리케이션을 구축한다.
- LangGraph는 상태(state) 기반으로 에이전트들을 디커플링하며, 의도 탐지·스키마 추출·텍스트-Cypher·쿼리 실행·요약 생성 에이전트를 유향 그래프로 연결한다.
- 법집행 수사 워크플로(범죄→ANPR 카메라→차량→소유자 전과)를 사례로 공간·시간·이력 분석을 통합한 맥락 인식 조사를 시연한다.

---

## 1. Building the LangGraph Pipeline

> LangGraph는 LLM 기반 상태유지(stateful)·다중행위자 애플리케이션을 위한 라이브러리다.

- 상태 기반 통신: 에이전트가 직접 데이터를 주고받지 않고 공유 상태(화이트보드)를 읽고 쓰며, 워크플로를 유향 그래프(각 노드=에이전트 함수)로 구현한다.
- 동적 엣지 해결(dynamic edge resolution)로 선행 에이전트 출력에 따라 다음 노드를 선택해 라우터부터 자율 시스템까지 다양하게 만든다.

### 15.1.1 ~ 15.1.3 Architecture, Configuration, Schema

- 백엔드 핵심은 LangGraph 워크플로이며, 설정 제공자(프롬프트·설정)·스키마 제공자(DB 스키마 추출)·질문 처리 인터페이스(이벤트 스트림)가 지원한다.
- 설정 컴포넌트는 Jinja2 템플릿과 KG 주석을 중앙 관리(notes·examples·prompts 섹션의 YAML)하여 유지보수·버전 관리를 쉽게 한다.
- 스키마 제공자는 apoc.meta.schema로 기술 스키마를 추출→skip 리스트로 필터링→설명으로 강화하여 LLM 친화적 개념 스키마로 변환(Node·Property·Relationship 데이터 모델)한다.

### 15.1.4 ~ 15.1.6 State, Agents, Integration

- AgentState(TypedDict)는 question·output_type·schema·query·results_error·summary·retries 등 파이프라인 진행 상태를 담는다.
- 파이프라인 에이전트: 의도 탐지(진입점, 시각화 타입 결정)→스키마 추출(Neo4jSchema 사용)→텍스트-Cypher(선택·주석 맥락 포함)→쿼리 실행(오류 처리·재시도)→후처리 동적 엣지(retry/summarize/END)→요약 생성.
- 쿼리 실행은 최대 3회 재시도하며, graph/map은 요약으로, table은 직접 종료로 라우팅한다.
- 통합 계층은 generator 함수(processQuestion)로 파이프라인 실행을 update·result·visualization 이벤트 스트림으로 변환해 실시간 프론트엔드 피드백을 제공한다.

```
  intent_detection -> schema_extraction -> text_to_cypher -> query_execution
                                                |
                          post_query_execution (conditional edge)
                          retry -> text_to_cypher
                          summarize -> generate_summary -> END
                          END
```

---

## 2. Streamlit Application

> Streamlit은 채팅 인터페이스·시각화·Python-우선 통합으로 시스템 요구사항에 잘 맞는다.

- 인터페이스: 그래프 캔버스(노드·관계 탐색·선택), Selection 컬럼(맥락 인식 질문 지원), 질문 입력, History 영역(맵·테이블 등 실시간 적응 표시).
- Streamlit의 session state와 자동 UI 갱신으로 이벤트 핸들링 메커니즘 없이 실시간 진행 상황을 반영한다.
- MessageHistory 객체가 상호작용 이력과 현재 파이프라인 상태를 유지하며, 임시 placeholder(실시간 피드백)와 영구 상태(END 이벤트 시 재렌더)를 결합한다.

---

## 3. Expert-Emulating Investigation

> 자연어 질의로 범죄·감시 카메라·차량 연결을 탐색하는 실제 수사 워크플로를 시연한다.

- 스키마: Crime-ANPRCamera(공간 관계)-CameraEvent(차량 탐지)-Vehicle-Person(소유자)-Crime(COMMITTED)으로 공간·시간·관계 분석을 결합한다.
- 초기 사건 식별: "수사 중인 범죄 노드 하나 반환" → 형사 침입(criminal trespass) 사건과 "EB"로 시작하는 검은 차량 단서를 식별.
- 공간 분석: "선택한 범죄에서 1km 내 ANPR 카메라" → 시스템이 자동으로 맵 시각화를 선택해 공간 관계를 표시.
- 차량 패턴 탐지: "선택한 카메라가 2023-06-15에 탐지한 검은색·EB 시작 차량" → 시간·외형·번호판 제약을 단일 쿼리로 통합.
- 맥락 인식 정제: 수사관 역할·의도를 제공하면 시스템이 노드 속성에서 제약을 자율 추출하고, 한 차량이 사건 시각 전후 두 번 탐지된 의심 패턴을 분석한다.
- 이력 분석: "이전 범죄자 소유 차량은?" → 의심 차량 소유자가 동종 전과(형사 침입)가 있음을 발견해 공간·시간·이력 증거를 일관된 수사 내러티브로 통합한다.

---

## 4. Future Directions and Enhancements

> 이 구현은 턴키 솔루션이 아닌 토대로, 관찰가능성(observability)과 전문가 에뮬레이션 아키텍처에 강점이 있다.

- 사용에서 학습: 모든 쿼리가 의도·패턴·효과 정보를 생성하며, "불만형" 질문을 페인 포인트 대시보드로 분류하고 성공 상호작용을 예시 DB에 반영한다.
- 핵심 역량 강화: 전문가처럼 예비 쿼리로 데이터 구조를 이해하는 스키마 강화 에이전트, 대규모 KG를 위한 다층(multilayer) 스키마 관리(상세→도메인 뷰).
- 고급 진화 경로: 순수 in-context learning의 확장성 한계를 넘어 스키마 자체를 훈련 데이터로 한 파인튜닝(연구상 in-context learning이 작업 특화 적응보다 일관되게 저조), 관찰가능 파이프라인 구조 유지하며 선택적 교체.

---

## Summary (핵심 정리)

- 전문가 에뮬레이션 접근은 인간 전문가의 그래프 DB 상호작용을 모방하며, LangGraph의 상태 기반 설계가 각 컴포넌트가 독립 추론하는 모듈식·관찰가능 AI 파이프라인을 만든다.
- 파이프라인 통합 아키텍처는 처리 업데이트를 대화형 UI로 실시간 스트리밍하고, 맥락 인식 쿼리 생성은 스키마 지식·사용자 선택·대화 이력을 결합한다.
- 다단계 분석 워크플로는 공간·시간·이력 분석을 단일 일관 프로세스로 통합하며, 메시지 이력 관리가 상태유지 대화와 실시간 진행 갱신을 가능하게 한다.
