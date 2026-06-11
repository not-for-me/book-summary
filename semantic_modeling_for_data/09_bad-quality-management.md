# 09. Bad Quality Management

## 챕터 개요 (3줄 요약)
- model 품질은 specification·개발 실수뿐 아니라 품질을 측정·관리하는 방식에서도 좌우된다.
- 품질을 trade-off 집합으로 다루고 risk·benefit에 연결하며, 올바른 metric을 써야 한다.
- vague assertion을 crisp하게 측정하거나 model 품질을 IE 품질과 동일시하는 것은 오류다.

---

## 1. Not Treating Quality as a Set of Trade-Offs
> 모든 dimension을 동시에 최대화할 수 없는 trade-off가 존재하며, 이를 무시·망각하는 것이 문제다.

### 1.1 Semantic Accuracy vs Completeness
> accuracy와 completeness는 상호 의존하지 않아, accuracy 통제를 느슨히 하면 completeness를 가속할 수 있다(위험한 유혹).

- 예: EU member state class에 127개국 넣으면 completeness 100%·accuracy 21%, 정확한 2개만 넣으면 accuracy 100%·completeness 7%. 자동 acquisition이 고정확·도메인이 작고 정적일 때만 trade-off 약화 → 전략적 결정 필요.

### 1.2 Conciseness vs Completeness
> 완성을 서두르면 redundant element를 탐지·제거할 시간이 없어 conciseness가 희생된다.

- 1만 term 추가 시 synonym 탐지·grouping을 건너뛰고 모두 distinct entity로 넣으면 더 complete하나 덜 concise. completeness가 subjective·context-dependent라 redundancy 판단도 어려움(저빈도 entity 제외 vs 불완전 주장). 해법: redundancy 탐지 메커니즘·기준·threshold 합의.

### 1.3 Conciseness vs Understandability
> redundancy 기준 없이 conciseness에 집착하면 human 이해를 해친다.

- 예: 비class entity에 정의 금지, 3단어 초과 이름 금지 → acronym(ADHD, DARE) 완전명 추가 불가. sweet spot 찾기가 까다로움.

### 1.4 Relevancy to Context A vs Context B
> 다중 시나리오에서 한 시나리오에 relevant하게 만들면 다른 시나리오엔 덜 유용·harmful할 수 있다.

- 예: app A는 풍부한 lexicalization(ambiguity 무관), B는 minimal ambiguity; A는 accuracy 우선, B는 completeness 우선; region A는 single truth, B는 다양한 관점; A는 복잡 axiom, B는 성능 위해 불가.
- 종종 context를 처음엔 모름 → 처음부터 잘 정의된 model 전략 필요(Ch10).
- **multiple models**: dimension별 버전 생성으로 일부 trade-off 해결 가능하나 유지 부담 증가 → maintenance를 outsource할 수 있을 때만(Textkernel은 client가 유지 책임질 때만 custom 버전).

---

## 2. Not Linking Quality to Risks and Benefits
> 품질을 risk·benefit 참조 없는 단순 metric 값으로 다루는 것은 무의미하다.

- 예: ESCO의 비strict synonym은 Textkernel엔 큰 문제지만 navigation 용도엔 OK. 95% accuracy biomedical ontology는 Watson엔 충분하나 의료 진단엔 위험. accuracy 10% 올렸는데 business 기여 0일 수 있음.
- 필요: ① 어떤 dimension이 성패에 기여하는지 알기(전략·specification 단계), ② 그 dimension·risk를 반영하는 맞춤 metric 선택(off-the-shelf 유혹 경계), ③ metric 값을 risk/benefit 추정에 연결(drug-disease 관계 최소 80% accuracy 필요 vs 15세기 아시아 event 저coverage는 OK).
- **harm 방지**: Amazon 채용 engine이 여성 차별(남성 이력서 패턴 학습) → 철회. 실제 사람에 영향 주는 model의 bias 미발견은 품질 관리의 대실패.

---

## 3. Not Using the Right Metrics
> 문헌의 metric이 항상 유용·적합한 건 아니며(context 밖 정의, model 이질성), scrutinize·적응해야 한다.

### 3.1 Metrics with Misleading Interpretations
> 실제 의미하지 않는 것을 의미한다고 제시되는 metric이 있다.

- **Inheritance Richness(IR)**(class당 평균 subclass 수): 저IR=상세, 고IR=일반이라 해석하나 — film genre를 subclass로(IR 높음) vs individual로(IR 0) 표현해도 같은 coverage. 구조적 metric으로 의미 품질 결론 내릴 때 주의(같은 의미의 다중 표현).

### 3.2 Metrics with Little Comparative Value
> class가 individual보다 중요하다고 가정하는 metric은 비교 가치가 작다.

- **Average Population(AP)**(individual/class): genre가 class면 AP=5,050, individual이면 100 — 같은 지식인데 첫 모델이 더 잘 채워진 듯. **Class Richness(CR)**도 동일 paradox(50% vs 100%).

### 3.3 Metrics with Arbitrary Value Thresholds
> risk/benefit 추정 없는 metric보다 나쁜 건 자의적 link다.

- **OQuaRE**(SQuaRE 기반): CROnto·WMCOnto·NOCOnto·TMOnto 등을 1~5 품질 점수에 매핑. "class당 parent 최대 8개 acceptable" 같은 threshold가 context 밖·자의적. 이런 표는 필요하나 value-score 매핑을 모든 stakeholder가 이해·합의해야.

### 3.4 Metrics That Are Actually Quality Signals
> 일부 metric은 품질이 아닌 좋/나쁠 **확률**을 정량화한다.

- **outlier 수**(accuracy metric로 제안): 모든 outlier가 틀린 건 아님 — 오류 탐지엔 쓰되 품질 보고엔 쓰지 말 것. **acquisition 방식 기반 trustworthiness 점수**(expert 수동=1, 자동 unstructured=0): expert가 늘 객관적이지 않고 manual이 늘 정확하지 않음. signal은 문제 탐지에 쓰되 품질 보고에 쓰지 않는다.

### 3.5 Measuring Accuracy of Vague Assertions in a Crisp Way
> vague assertion의 진위는 subjective·context-dependent라 단순 참 비율 계산은 오도한다.

- 4 실수: ① context 없이 판정 요청(judge가 자의적 context 사용), ② assertion당 judge 1명(일방적·disagreement 모름), ③ 다수 judge에 **consensus 강요**(가벼운 불일치엔 OK지만 controversial하면 consensus bias — 논의로 model 개선에만 활용), ④ disagreement 미보고(전원/과반/1명 등 기준만 쓰고 risk 정량화 안 함).
- disagreement는 vagueness 외에 ambiguity·지식 부족·bias·fatigue도 원인 → vagueness 관련만 isolate·정량화.

### 3.6 Equating Model Quality with Information Extraction Quality
> IE 도구 효과가 model 품질의 지표지만, model이 도구만큼 좋/나쁘다고 가정하면 오도된다.

- model 품질은 source 품질에도 의존(부정확 source면 도구가 좋아도 나쁨). 예: Zaveri의 DBpedia 11.93% 부정확은 "Wikipedia에서 정확히 추출됐나"를 물은 것 — 실제 오류는 더 높을 수 있음(Wikipedia 100% 정확이 아닌 한).
- 측정 방식 차이: NER 70% precision·60% recall로 Location class 채울 때 — model엔 entity 1개만 필요하므로 몇 번 occurrence를 봐야 추가할지에 따라 오류 확률 달라짐(1회면 70%, 10회면 훨씬 낮음). NER recall은 corpus 기준, model completeness는 도메인 기준 — 100% recall이어도 corpus가 도메인 전체를 안 담으면 model 불완전.

---

## Summary (핵심 정리)
- 품질을 risk·benefit에 연결한다 — 아니면 무의미한 숫자일 뿐이다.
- 품질을 trade-off로 다루고 무엇이 더 큰 benefit을 주는지 결정한다.
- quality signal은 수집하되 metric으로 쓰지 않는다.
- actionable한 정보를 주는 metric을 쓴다.
- model이 누구에게도 harm을 주지 않도록 품질을 관리한다.
