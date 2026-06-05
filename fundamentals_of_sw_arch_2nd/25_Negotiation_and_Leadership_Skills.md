# 25. Negotiation and Leadership Skills

## 챕터 개요 (3줄 요약)

- 협상·리더십은 효과적 아키텍트의 핵심 하드 스킬로, 거의 모든 결정이 도전받기 때문이다.
- 비즈니스 이해관계자·다른 아키텍트·개발자와의 협상 기법을 상황별로 제시한다.
- 아키텍트는 4C(소통·협업·명료·간결)와 실용적이면서 비전 있는 리더십으로 존경을 얻는다.

---

## 1. Negotiation and Facilitation (협상과 촉진)

> 아키텍트의 결정은 개발자·다른 아키텍트·이해관계자에게 늘 도전받으므로 협상력이 필수다.

### 비즈니스 이해관계자와 협상

시나리오: SVP Parker가 'five nines(99.999%)' 가용성 고집(실제론 99.9%면 충분). 기법: (1) 버즈워드·전문용어의 단서 주목("어제 필요했다"=출시 기간 중요), (2) 사전 정보 최대 수집('나인즈'를 연간 다운타임 시간·분으로 환산해 대화 — 99.9%=일 86초), (3) 최후수단으로 비용·시간으로 표현(처음부터 꺼내지 말 것), (4) 분할 정복(divide and conquer — 전체가 아닌 특정 영역만 five nines 필요한지 좁히기).

```
  99.9%   (3 nines) = 8h 46m/year  (~86 sec/day)
  99.999% (5 nines) = 5m 35s/year  (~1 sec/day, costly)
```

### 다른 아키텍트와 협상

시나리오: Addison이 메시징 대신 REST 고집(Google·AI 검색 근거). 기법: 시연이 토론을 이긴다(demonstration defeats discussion — 운영 유사 환경에서 비교). 과하게 논쟁적·개인적이지 말고, 차분한 리더십과 명료한 추론. 격해지면 협상 중단 후 재개.

### 개발자와 협상

Ivory Tower 안티패턴(위에서 명령) 회피. 기법: (1) 명령 대신 정당화 제공(이유를 먼저 말해야 끝까지 들음, "you must" 대신 "this means"), (2) 개발자가 스스로 해법에 도달하게 함(Framework Y로 보안 요구 충족 시연해보라 — 실패하면 X 수용 buy-in, 성공하면 더 나은 해법).

---

## 2. The Software Architect as a Leader (리더로서의 아키텍트)

> 효과적 아키텍트의 약 50%는 촉진·리더십 같은 대인 기술이다.

### The 4 Cs of Architecture

Essential complexity("어려운 문제다") vs Accidental complexity("문제를 어렵게 만들었다" — 자기 가치 증명·job security 위해 추가하면 존경 상실). 이를 피하는 4C: Communication, Collaboration, Clear, Concise.

### Be Pragmatic, Yet Visionary (실용적이면서 비전 있게)

비전가(전략적·미래 계획)와 실용가(예산·시간·팀 역량·트레이드오프·기술 한계 고려)의 균형. 사용자 급증 시 비전가는 복잡한 data mesh를, 실용가는 먼저 병목 식별·캐싱.

### Leading Teams by Example (모범으로 이끌기)

직급이 아닌 모범으로(대위-병장 비유). "사람 문제다"(Weinberg). "그건 멍청한 생각"은 협업 중단. "what you need to do is" 대신 "have you considered"로 질문화. 요청을 부탁으로 전환(이름 사용, "곤란한 상황이라 도와달라"). 첫 만남엔 악수+눈맞춤(문화 차이·개인 경계 존중, 포옹 금지). 'go-to person'이 되고 brown-bag 세션 주최.

---

## 3. Integrating with the Development Team (팀과 통합)

> 회의로 가득 찬 일정 속에서 팀과 함께할 시간을 확보해야 한다.

회의 유형: 남이 부른 회의(통제 어려움 — 왜 필요한지·의제 요청, 일부만 참석, 테크리드 대신 참석), 내가 부른 회의(최소화·의제 준수). 개발자 flow state(몰입 상태) 방해 금지 — 회의는 아침·점심 후·하루 끝에. 온사이트면 팀과 함께 앉거나 돌아다니며 보이기. 원격은 협업이 더 어려움.

---

## Summary (핵심 정리)

- 협상 기법(정보 수집·분할 정복·시연·정당화)으로 이해관계자·아키텍트·개발자와 합의를 만든다.
- 4C와 실용-비전 균형, 모범 리더십(질문화·부탁 전환·이름 사용)으로 존경을 얻는다.
- 회의를 통제해 팀과 통합하고 flow state를 존중하며 'go-to person'이 된다.
