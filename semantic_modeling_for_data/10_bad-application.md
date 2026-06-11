# 10. Bad Application

## 챕터 개요 (3줄 요약)
- 같은 도메인용 model이라고 해서 그 semantics가 application에 직접 유용하다고 가정하는 것이 핵심 pitfall이다.
- entity resolution에서는 model의 일부 element가 도움이 아니라 disambiguation을 오히려 harm할 수 있어, ambiguity 유형과 model의 evidential adequacy를 측정해 pruning·enrich해야 한다.
- semantic relatedness는 context 밖에선 합의가 쉽지만 context 안에선 어려운 vague relation이므로 "should be related"를 구체·context-specific하게 변환해야 한다.

---

## 1. Bad Entity Resolution
> entity resolution은 텍스트의 entity mention을 탐지해 model의 entity로 매핑하는 task다.

- 핵심 문제는 **ambiguity**(한 term이 여러 entity 지칭 — "SPAWN"=영화/comic/생물). 예: Steel 리뷰에서 배우·영화 entity를 DBpedia에 매핑.

### 1.1 How Entity Resolution Systems Use Semantic Models
> 4 입력(텍스트, target entity, entity thesaurus, 맥락 증거 지식 자원)으로 2단계 작동한다.

- Miller·Charles의 **strong contextual hypothesis**(유사 의미 term은 유사 context에 출현). 맥락 = 주변 term 또는 model의 관련 entity(영화 disambiguation context에 starring 배우).
- 2단계: ① thesaurus로 candidate entity 추출, ② 맥락 증거로 disambiguate(annotated 텍스트면 context 유사도, semantic model이면 graph 유사도).

### 1.2 When Knowledge Can Hurt You
> 효과는 텍스트 내용과 증거 data의 정렬도에 의존 — 도메인 밖이거나 텍스트에 없는 element는 harm한다.

- 예: "Ronaldo scored for Real Madrid"(2015) — 현재·과거 선수 둘 다 고려하면 disambiguate 실패. 현대 경기 텍스트엔 former player 관계가 안 나오므로 무시하는 게 정확. 영화 리뷰에서 DBpedia의 비배우 person entity를 제거하면 precision 향상(Roger Moore 배우 vs 과학자).

### 1.3 How to Select Disambiguation-Useful Knowledge
> precision/recall로 측정하고, ambiguity 유형과 model의 evidential adequacy를 진단한다.

- **저precision** 원인: 높은 ambiguity, 부적절한 맥락 지식. **저recall** 원인: 불완전한 thesaurus(surface form 누락), 최소 증거 요구 미충족.
- **ambiguity 측정 5유형**: lexical(entity 아닌 일반어 혼동 — Factual), target-to-target(Tripoli 그리스/리비아), target-to-nontarget(Barcelona 팀/도시), nontarget-to-target, global(thesaurus·model 밖 — Apple 회사/과일). 텍스트 표본을 수동·자동 annotate해 각 유형 % 측정.
- **evidential adequacy**: model richness(entity당 relation/attribute 값 — 비관련 target % , 평균 관련 entity 수, 특정 relation당 평균)와 텍스트 prevalence(증거가 텍스트에 실제 등장하는지). model 쿼리 + 텍스트로 측정.

### 1.4 Improving Disambiguation Capability
> metric 값 → 진단 → action(Table 10-2).

- 높은 lexical → WSD 컴포넌트 개선. 높은 global → domain classifier로 무관 텍스트 필터. 높은 target-to-nontarget·낮은 nontarget-to-target → **evidence model pruning**(target과 무관 entity 제거 → prevalence 낮은 relation 제거). 낮은 richness → prevalent relation부터 enrich. 높은 richness·낮은 prevalence → 텍스트에 나올 entity로 교체·확장. 낮은 prevalence·낮은 ambiguity → 최소 증거 threshold 낮춰 recall↑. **action 후 새 test set으로 재측정**.

### 1.5 Two Entity Resolution Stories (Knowledge Tagger, iSOCO)
> 진단 framework를 두 사례에 적용해 효과를 높였다.

- **축구 선수**(스페인 Liga 하이라이트): DBpedia로 precision 60%·recall 55%. 진단 — target ambiguity 30%·target-to-nontarget 56%, nontarget-to-target 4%. 비축구 entity와 무가치 relation pruning(현재 club·co-player·manager만 유지, prevalence 85~95%) → precision 82%·recall 80%.
- **뉴스 기사 속 startup**: 4,000 회사 KG로 precision 35%·recall 50%. 진단 — global ambiguity 40%·lexical 10%(Factual·Collective·Prime). domain classifier(90% accuracy)로 필터 → precision 72%, 대문자 휴리스틱 추가 → 78%/57%, 증거 threshold 낮춤 → recall 62%.

---

## 2. Bad Semantic Relatedness
> semantic model의 또 다른 흔한 용도는 term·entity 간 relatedness 계산이다.

- 옵션: 기존 relatedness relation 재사용(WordNet, Eurovoc), 기존 measure 적용(최단 경로 길이), 자체 measure 정의(broader=유사, narrower=비유사), 외부 data에서 추출, 조합. 핵심 pitfall: **app·user가 필요로 하는 것과 다른 relatedness 계산**.

### 2.1 Why Semantic Relatedness Is Tricky
> context 밖에선 합의가 쉽지만 context 안에선 어려운 vague relation이다.

- 예: Prolog·Python은 둘 다 언어라 관련되나, Prolog 경력자를 Python 개발자로 채용? No. 독일어와 가까운 언어 — 언어학자는 Dutch/Danish, 독일 시장 recruiter는 English/French.
- 같은 history ontology를 서점(구매)과 도서관(대출) 검색에 적용 — 도서관 사용자는 더 generic event(WWII) 책도 OK(무료), 서점 사용자는 한 chapter만 다룬 책은 안 삼.

### 2.2 How to Get the Semantic Relatedness You Really Need
> "semantics-first" 접근으로 사용자의 가정·기대를 명시적·context-specific하게 만든다.

- "should be related"를 구체화: 관련될 듯한 예시를 judge하게 하고 이유를 묻되, 개별 예시 micro-argue 말고 pattern·rule 식별. 예: 서점="A에 관심 있는 사용자가 B 책도 사고 싶어함", 도서관="A 정보 찾는 사용자가 B 책도 유용".
- 순서: ① 기존 model의 relatedness relation 확인, ② 다른 기존 model에 있는지, ③ 표준 measure 테스트, ④ custom metric/rule, ⑤ 외부 source에서 mining. 잘못된 relatedness를 노리면 무엇을 해도 실패.

### 2.3 A Semantic Relatedness Story (HTSO 전력시장, 2008)
> 그리스 전력시장 ontology + semantic search로 query expansion.

- 사용자 interpretation 도출 — "시장 참여에 필요한 모든 정보를 중요도 순으로". relation별 weight(-1~1) 기반 custom measure: isImportantPartOfProcess·isInterestedInProcess=1, performsAction=0.8, isPerformedByParticipant=-0.5, hasObligation/hasRight=0.5.
- 25 query golden set으로 표준 distance 기반 vs custom 비교 → custom이 precision·recall 약 8% 향상.

---

## Summary (핵심 정리)
- model의 모든 relation·aspect가 disambiguation에 도움 되는 건 아니며, 일부는 harm한다.
- disambiguation 유용성을 높이려면 model 유용성과 ambiguity 유형을 측정해 action을 도출한다(Table 10-2).
- semantic relatedness는 context 밖에선 합의가 쉽지만 안에선 어려워 까다롭다.
- app이 실제 필요로 하는 relatedness를 얻으려면 "should be related"를 구체·context-specific하게 변환한다.
