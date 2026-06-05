# 15. The AI-Powered Software Engineer

## 챕터 개요 (3줄 요약)

- Jacquard loom이 직조공을 대체하지 않고 변화시켰듯, AI는 개발자를 대체하지 않고 superpower를 주어 더 빠르게 만들게 한다.
- AI의 핵심 개념(ML, deep learning, GenAI, LLM)과 강점·한계를 이해하고, AI를 pair programmer로 활용한다.
- prompt engineering을 마스터하고, vibe coding의 위험을 인식하며, 코드 작성에서 problem-solving으로 초점을 옮긴다.

---

## 1. What Is AI Really?

> AI는 규칙 기반 시스템부터 신경망까지 포괄하는 광범위한 분야다.

- AI는 새롭지 않으며 1950년대부터 추구됐고, AI winter를 거쳐 2022년 11월 ChatGPT로 대중화됐다(5일 만에 100만 사용자).
- 전통 프로그래밍은 단계별 명령을 작성하지만, 현대 AI는 패턴/확률을 인식해 통계적으로 가장 가능성 높은 답을 선택한다.
- ML(Machine Learning): 데이터에서 패턴을 학습한다("input+output=program") — classification, regression, clustering.
- Deep Learning: 다층 neural network를 쓰는 ML의 하위집합으로 복잡한 패턴과 뉘앙스를 포착한다.
- GenAI(Generative AI): 기존 데이터 분석을 넘어 새로운 콘텐츠(text/code, image, audio, video)를 생성한다.
- LLM(Large Language Model): deep learning + GenAI로 맥락과 뉘앙스를 이해한다(ChatGPT, Claude, Copilot) — temperature가 무작위성을 제어한다.

```
AI (umbrella)
 └ Machine Learning
    └ Deep Learning (neural networks)
       └ Generative AI
          └ LLMs (ChatGPT, Claude, Copilot)
```

## 2. Understanding AI's Capabilities and Limitations

> AI를 효과적으로 쓰려면 강점과 한계를 모두 알아야 한다.

- AI가 잘하는 것: 반복 코딩, 자동화/glue 스크립트, 코드 설명, 문서 생성, refactoring 제안, test case 생성, 코드 변환, UI mockup, 언어·framework 기능 이해.
- AI의 한계: 실시간 지식 없음, hallucination(자신감 있는 오답), codebase 맥락 부족, 학습 데이터의 편향, business 요구 이해 불가.
- 도메인 간 일관성 없는 성능(niche 기술에 약함)과 privacy·security 위험(proprietary 코드 공유 주의).
- model과 product를 구분하라 — 제품이 web 검색 같은 tool로 한계를 보완할 수 있다.
- AI 생성 코드는 인간 코드와 동일한 엄격한 프로세스(테스트, code review)로 다뤄라 — first draft로 간주하라.

## 3. AI as Your Pair Programmer

> AI는 24/7 사용 가능한 궁극의 pair programming 파트너가 될 수 있다.

- Standalone Chatbot(ChatGPT, Gemini, Claude): 개념 이해·디버깅·architecture 결정에 좋다 — 맥락·persona를 제공하라.
- Inline IDE Assistant(GitHub Copilot, JetBrains AI, CodeWhisperer): 실시간 제안·자동완성으로 반복 코드·test·문서에 좋다 — 수락 전 검토하라.
- Agentic AI IDE(Cursor, Junie, Cline): codebase 전체 맥락을 이해해 여러 파일에 걸친 변경을 한다(analysis → planning → implementation).
- 명확하고 구체적인 요구사항으로 시작하고, nonfunctional requirement를 미리 정의하며, 모든 변경을 신중히 검토하라.
- "you are the pilot, not the passenger" — 생성 여부와 무관하게 모든 코드에 책임을 진다.
- AI 코드를 수락하고 넘어가지 말고 "이 코드가 무엇을 왜 하는가"를 설명할 수 있어야 한다.

## 4. Prompt Engineering Fundamentals

> prompt engineering은 AI와 효과적으로 소통하는 법을 배우는 것이다.

- prompt 품질이 응답 품질과 직결된다(맥락 없는 "fix the login bug" vs 구체적 설명).
- 명확한 소통: 구체적이어야 한다("sorting" vs "ArrayList of Employee를 salary 내림차순 정렬, null 처리 포함").
- 구조가 성공을 결정한다: persona·길이·형식·예시를 명시하라.
- 명령이 아닌 가르침으로 접근하라 — chain-of-thought prompting으로 사고 과정을 안내한다.
- 즉시 적용 팁: 맥락 제공, 예시 사용, 환경 명시, role 지정, 반복·개선, AI에게 prompt 개선 요청, 좋은 prompt 저장.

### Advanced Techniques

- structuring: zero-shot(예시 없음), one-shot(예시 1개로 패턴 확립), few-shot(여러 예시로 분류 학습).
- organizational: XML tag/Markdown/JSON으로 task·requirement·constraint를 분리, task decomposition으로 복잡한 문제를 단계로 분할.

## 5. How AI Might Shape Software Engineering — Will AI Take My Job?

> AI는 개발자를 대체하지 않지만, AI를 쓰는 개발자가 안 쓰는 개발자를 대체한다.

- software engineering은 코딩 그 이상이다: 모호한 요구를 시스템으로 만들고, trade-off 판단("it depends"), 정치 탐색, mentoring, 비기술 stakeholder 소통.
- AI는 회사의 business model, 사용자의 고유 니즈, CEO의 "간단한 요청"이 왜 악몽인지 이해하지 못한다.
- Jacquard loom처럼 기술을 운영하고 활용하는 직조공이 번성했다.
- AI는 지루한 작업(boilerplate, 반복 test, 문서)을 없애 창의적·전략적 작업에 시간을 쓰게 하는 해방이다.
- 모든 기술 발전(assembly→고수준 언어, IDE, CI/CD)에도 개발자 수요는 매년 늘었다.
- 가치는 for loop를 쓰는 능력이 아니라 언제·왜 쓰고 큰 그림에 어떻게 맞추는지 아는 데 있다.

## 6. Vibe Code Reviews & AI as Force Multiplier

> vibe coding은 Andrej Karpathy가 풍자적으로 만든 용어로, 문제적 관행을 강조한다.

- vibe coding: agentic IDE로 prompt를 통해 전체 애플리케이션을 생성하는 것 — "throwaway weekend project"에 적합하다.
- 개인 expense tracker(버그=부정확한 예산) vs 기업 payroll 시스템(버그=급여 오류·법적 문제)의 위험은 천지차이다.
- "25% of code is AI-written" 헤드라인의 실제: 개발자가 issue 작성 → AI가 PR 생성 → 인간이 검토·merge(인간이 품질·결정 통제).
- code review가 그 어느 때보다 중요하다 — Neal Ford는 "tsunami of bad code"를 경고한다.
- AI가 생성해도 "내" 코드이며 PR 전에 왜 존재하고 무엇을 하는지 설명할 수 있어야 한다("AI 탓"은 변명이 안 됨).
- force multiplier: $5,000 카메라가 사진가를 만들지 않듯 AI가 더 나은 개발자를 만들지 않으며 기존 스킬을 증폭한다.
- 전통적으로 70% 시간을 boilerplate에, 30%를 problem-solving에 썼다면, AI로 이를 뒤집어 architecture·요구 분석·복잡한 business 문제에 집중한다.
- lines of code는 의미 있는 생산성 척도가 아니다 — "클라우드 비용 20% 절감"이 더 설득력 있다.

---

## Summary (핵심 정리)

- AI는 개발자를 대체하지 않지만 효과적으로 활용하는 개발자가 큰 우위를 가지며, AI는 replacement가 아닌 pair programming 파트너로 가장 잘 작동한다.
- AI의 강점(반복 작업, 코드 설명, 문서, 패턴 인식)과 한계(실시간 지식, hallucination, 맥락·보안)를 이해하고 인간 코드와 동일한 엄격함으로 다뤄라.
- 초점이 코드 작성에서 problem-solving으로 옮겨가 역할이 격상되며, prompt engineering을 익히고 모든 코드의 소유권을 유지하는 개발자가 번성한다.
