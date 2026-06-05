# 09. Creating Single- and Multi-Agent Systems

## 챕터 개요 (3줄 요약)

- 자율 에이전트(autonomous agent)는 LLM을 핵심으로 프로파일링·메모리·계획·행동 모듈로 구성된다.
- HuggingGPT처럼 LLM이 도구(모델)를 오케스트레이션하고, 다중 에이전트로 복잡한 작업을 해결한다.
- 법률·화학·의료 응용과 SaaS·MaaS·DaaS·RaaS 같은 새로운 비즈니스 패러다임을 다룬다.

---

## 1. Introduction to autonomous agents

> 자율 에이전트는 환경을 지각하고 추론·결정·행동을 인간 개입 없이 수행하는 시스템으로 AGI를 향한 단계다.

- LLM은 연역·귀납·가추(abductive) 추론과 단계별 추론으로 작업을 분해하고 행동을 조정한다.
- 프로파일링 모듈(profiling)은 역할(페르소나)을 정의하며 수작업 또는 LLM 자동 생성 방식이 있다.
- 메모리 모듈(memory)은 단기(컨텍스트)와 장기(외부 RAG/DB) 하이브리드로 읽기·쓰기·반성(reflection)을 수행한다.
- 계획 모듈(planning)은 작업을 분해하며 피드백 없는(CoT, ToT)과 피드백 있는(ReAct) 방식이 있다.
- 행동 모듈(action)은 계획을 실행하며 사전학습 지식이나 외부 도구(API·모델·DB)를 사용한다.
- 에이전트 개선: 파인튜닝(WebShop, EduChat, ToolBench)이나 프롬프트 엔지니어링을 사용한다.
- Toolformer는 자기지도 방식으로 도구를 API 호출로 다루며 모델의 일반성을 보존한다.

### 자율 에이전트 4모듈

```
Profiling -> define role/persona
Memory    -> short-term (context) + long-term (RAG/DB)
Planning  -> decompose task (CoT/ToT/ReAct)
Action    -> execute via pre-trained knowledge or tools
```

---

## 2. HuggingGPT and other approaches

> HuggingGPT는 언어를 인터페이스로 LLM이 전문 모델들을 오케스트레이션해 복잡한 멀티모달 작업을 푼다.

- HuggingGPT는 LLM(ChatGPT)을 두뇌로 삼아 Hugging Face 모델들을 조율하며 추가 학습이 없다.
- 4단계: 작업 계획(task planning), 모델 선택(model selection), 작업 실행(task execution), 응답 생성(response generation).
- 작업 계획은 task ID, type, dependencies, arguments 슬롯을 채우는 슬롯 필링으로 수행된다.
- 모델 선택은 작업 유형 필터링 후 다운로드 수로 랭킹해 top-k를 프롬프트에 넣는다.
- 실행은 하이브리드 추론 엔드포인트로 병렬화하며 <resource> 토큰으로 의존성을 처리한다.
- 한계: 효율(다중 LLM 호출), 계획 능력, 컨텍스트 길이, 불안정성(환각).
- ChemCrow는 화학 특화 도구를 Thought-Action-Action Input-Observation 형식으로 조율한다.
- SwiftDossier는 RAG와 에이전트로 신약 개발의 환각을 줄이고, ChemAgent는 동적 라이브러리로 지속 학습을 모방한다.

---

## 3. Multi-agent system

> 다중 에이전트는 서로 다른 전문성을 가진 LLM 에이전트들이 협력·비평하며 복잡한 작업을 해결한다.

- 법률: Chatlaw는 지식 그래프와 다중 에이전트로 법률 사무소 협업을 모방한다.
- 다중 판사 시스템(Hamilton)은 각 에이전트가 판사 역할을 하고 다수결로 의견을 도출한다.
- 의료/과학: AI Scientist는 아이디어 구상부터 논문 작성까지 전 과정을 수행한다.
- Virtual Lab은 PI(Principal Investigator)가 이끄는 이종 에이전트들이 단체·개별 회의로 연구한다(인간 개입).
- Virtual Lab은 SARS-CoV-2 KP.3 변이 항체 설계를 실험적으로 검증했다.
- Agent Laboratory는 문헌 검토·실험·보고서 작성 3단계를 자율 수행하며 인간 피드백을 받는다.
- 한계: LLM의 진짜 추론 부재로 인간 피드백 의존, 코드 오류, 높은 연산 비용.

---

## 4. Working with HuggingGPT

> HuggingGPT는 로컬 클론(모델 다운로드)이나 웹 서비스로 사용하며 모든 모델은 추론으로만 동작한다.

- 로컬 방식은 저장소(Jarvis)를 클론해 모델을 다운로드하고 로컬에서 실행한다.
- 웹 서비스 방식은 실행을 서비스에서 수행하며 OpenAI·Hugging Face 토큰이 필요하다.
- Git LFS(Large File Storage)는 큰 바이너리(모델 가중치, 확산 모델)를 포인터로 관리해 저장소를 가볍게 유지한다.
- macOS에서는 brew install git-lfs로 설치한다.
- 두 방식 모두 웹 기반 GUI를 지원한다.

---

## 5. SaaS, MaaS, DaaS, and RaaS

> LLM의 등장으로 소프트웨어·모델·데이터·결과를 서비스로 제공하는 새로운 비즈니스 패러다임이 열린다.

- SaaS(Software as a Service)는 지난 30년을 지배한 인터넷 혁명기 패러다임이다.
- MaaS(Model as a Service)는 ML·AI 모델을 제공해 인프라·전문성 투자를 줄여준다.
- DaaS(Data as a Service)는 정보 기반 의사결정을 위한 양질의 데이터를 제공한다.
- RaaS(Results/Outcome as a Service)는 모델의 출력(결과)만 제공해 결과 기반 과금을 한다.
- 사용자 이점: 낮은 초기 비용, 인프라 불필요, 빠른 시장 출시.
- 제공자 도전: 높은 개발 비용, 공정성·규정 준수, 모델 재학습 비용.
- 패러다임 선택은 자원·요구에 따라 다르며 다중 에이전트 시스템 기술 선택에 영향을 준다.

---

## Summary (핵심 정리)

- LLM은 계획·추론에 강하나 실행은 약해 전문 모델을 도구로 호출해 작업을 완수함을 배웠다.
- HuggingGPT 같은 단일/다중 에이전트로 산업 응용(법률·화학·의료)을 자동화하는 법을 익혔다.
- SaaS·MaaS·DaaS·RaaS 비즈니스 패러다임을 다뤘으며, 다음 장에서 에이전트 생태계를 다룬다.
