# 11. The Future Ahead

## 챕터 개요 (3줄 요약)

- AI 에이전트가 의료(AI scientist)·로보틱스·게임·웹 등 다양한 산업을 혁신하는 미래 전망을 다룬다.
- 인간-에이전트 소통, 다중 에이전트 우월성 논쟁, LLM의 추론·창의성 한계 같은 미해결 과제를 살펴본다.
- 기계적 해석 가능성, AGI로 가는 길, 윤리적 위험까지 폭넓게 논의한다.

---

## 1. AI agents in healthcare

> AI scientist 패러다임은 LLM과 도구를 결합해 생의학 연구를 자율적으로 수행하는 시스템을 지향한다.

- AI scientist는 문제를 하위 작업으로 분해하고 자율적으로 해결해 발견을 가속하는 것을 목표로 한다.
- 1990년대부터 데이터 기반 모델 패러다임이 발전했고, AlphaFold2는 PDB의 수백만 구조 덕에 가능했다.
- 생의학 AI 에이전트는 단일 LLM에 역할 프롬프트, 명령어 튜닝, in-context learning으로 전문성을 부여한다.
- 다중 에이전트는 브레인스토밍·비평·토론·협력으로 연구 가설을 다각도로 평가한다.
- Gao의 자율성 레벨: Level 0(도구), Level 1(연구 보조), Level 2(협력자), Level 3(과학자).
- 현재는 Level 1까지만 존재하며 Level 2·3에는 새 아키텍처가 필요하다.
- ChemCrow는 18개 도구를 통합해 곤충 기피제·유기촉매를 자율 합성한 Level 1 예시다.

---

## 2. AI agents in other sectors

> LLM 에이전트는 로보틱스, 게임, 웹 자동화 등 다양한 산업에 전 지구적 영향을 미친다.

- 물리적(physical) 에이전트는 중력·마찰·관성 같은 물리 법칙을 이해하며 환경과 상호작용한다.
- 물리적 에이전트의 이점: 자연어 인간 상호작용, 유연성·적응성, 멀티모달 능력.
- 물리적 에이전트의 과제: 데이터셋·학습, 로봇 구조, LLM 배포(클라우드 의존), 보안.
- 게임 에이전트는 RL과 CoT로 게임 환경·캐릭터와 상호작용하며 전략을 추론한다(Pokémon 사례).
- 웹 에이전트는 지각(HTML/스크린샷), 추론(LLM), 웹 상호작용 모듈로 반복 작업을 자동화한다.
- PaLM-SayCan, PaLM-E는 LLM으로 로봇을 제어한 초기 실험이다.

---

## 3. Challenges and open questions

> 안전한 AI 에이전트 사용을 위해 인간 소통, 다중 에이전트, 추론·창의성, 해석성, AGI, 윤리 과제가 남아 있다.

- 인간-에이전트 소통은 투명성(transparency)과 통제(control) 두 원칙에 기반해야 한다.
- 소통 과제: 명확한 목표 획득, 사용자 선호 존중, 피드백 반영, 에이전트 능력·행동·진행·결과 전달.
- 다중 에이전트(MAS)는 단일 에이전트 대비 비용·지연·통신 오버헤드가 크나 성능 향상은 미미할 수 있다.
- MAS 실패 원인 3그룹: 명세·설계 실패, 에이전트 간 불일치, 작업 검증·종료 실패.
- 추론 한계: LLM은 패턴 매칭 기계로 토큰 편향(token bias)과 프롬프트 민감성을 보인다.
- GSM-Symbolic 연구는 수치 변경 시 성능이 급락해 형식적 추론(formal reasoning) 부재를 시사한다.
- CoT(Chain-of-Thought)의 이점은 주로 수학·기호 추론에 국한되며 외부 도구가 필요하다.
- 창의성: LLM 출력은 인터넷 텍스트에 매핑 가능해 인간보다 창의성이 낮다(DJ vs 작곡가 비유).
- 기계적 해석 가능성(mechanistic interpretability): SDL·SAE로 다의적(polysemantic) 뉴런의 희소 표현을 분해한다.
- AGI(Artificial General Intelligence)로 가는 길: 스케일링만으로는 부족하며 세계 모델(world model)·체화(embodiment)가 논의된다.

### 생의학 에이전트 자율성 레벨

```
Level 0: tool (no autonomy)
Level 1: research assistant (orchestrator, human-defined tasks)
Level 2: collaborator (refines hypothesis)
Level 3: scientist (generates novel hypotheses, end-to-end)
```

### MAS 실패 3범주

```
1. Specification & system design failures
2. Inter-agent misalignment
3. Task verification & termination failures
```

---

## 4. Ethical questions

> AI 에이전트는 의인화, 과도한 영향, 악용, 경제·환경 영향 같은 윤리적 위험을 수반한다.

- 완전 자율 AI 에이전트는 개발하지 말아야 한다는 주장도 있으며 인간 판단이 중요하다.
- 의인화(anthropomorphizing) 위험: LLM은 의식·감정이 없으나 감정을 모방해 잘못된 신뢰를 유발한다.
- 영향 위험: 설득(persuasion), 조작(manipulation), 기만(deception), 강압(coercion), 착취(exploitation).
- 악용: 허위정보 대량 생성, 피싱·사이버공격·사기, 권위주의 정부의 감시.
- 경제 영향: 고용(약 47% 자동화 위험), 일자리 질, 불평등 심화.
- 환경 영향: 하드웨어 제조, 학습·추론의 에너지 소비와 CO2 배출 증가.
- 추론 모델은 과잉 사고(overthinking)와 과소 사고(underthinking) 문제를 보인다.

---

## Summary (핵심 정리)

- 에이전트가 의료·로보틱스·게임·웹 등 산업을 혁신하는 미래 전망을 살펴봤다.
- LLM의 추론·창의성 한계, 다중 에이전트 실패, 해석 가능성, AGI 과제를 이해했다.
- 기술적·윤리적 도전이 여전히 남아 있으며, 책은 이 혁명을 이해할 도구를 제공했다.
