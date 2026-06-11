# 04. Semantic Model Quality

## 챕터 개요 (3줄 요약)
- semantic model의 품질은 application-centered(특정 app 개선 측정)와 application-neutral(도메인·data 대비 품질 측정) 두 접근으로 평가한다.
- 핵심 품질 dimension: semantic accuracy, completeness, consistency, conciseness, timeliness, relevancy, understandability, trustworthiness.
- 각 dimension마다 측정 metric·방법과 품질 저하의 흔한 원인이 있으며, 모든 dimension에서 고품질 달성은 매우 어렵다.

---

## 0. 평가 접근 두 가지
> application-centered는 app 효과 향상을 직접 보고, application-neutral은 도메인·data 대비 일관·전이 가능한 품질을 본다.

- application-centered: app에 model 적용 전후 비교. 즉각적 fitness 확인 가능하나, 효과 없음이 곧 model 저품질은 아니고(app 사용법 문제 가능), 무엇이 틀렸는지 모르며, 여러 app에 동시 사용 시 한쪽 개선이 다른 쪽을 악화시킬 수 있음.
- application-neutral: 같은 data 쓰는 app 간 일관·전이. 본 챕터의 dimension은 relevancy를 제외하고 모두 application-neutral.

---

## 1. Semantic Accuracy
> model의 semantic assertion이 참으로 받아들여지는 정도다.

- 부정확 원인: ① 자동 IE 방법의 부정확(가장 흔함 — 2018 SemEval hypernym discovery 최고 성능도 medical 36%·music 44%), ② source data 자체 오류(Wikipedia 2.8% 틀림, wiki vandalism), ③ modeling element 의미 오해(가짜 synonym, instance를 class로 — Ch7), ④ domain 지식·전문성 부족(Ch8), ⑤ vagueness(한 집단엔 참, 다른 집단엔 거짓).
- 측정: statement 표본을 human judge(전문가·사용자·crowd)에게 참/거짓 판정. 다수 judge + inter-agreement(특히 vague statement). 자동 탐지: 통계적 outlier 탐지, consistency rule·axiom 위반 reasoning(단 충분히 axiomatize·기존 오류 적어야).
- **주의: inferred inaccuracy** — 잘못된 "A⊂B" 하나가 A의 instance 1만 개에 대해 1만 개의 잘못된 statement로 증식.

---

## 2. Completeness
> 모델에 있어야 할 element가 실제로 있는 정도다.

- **schema completeness**(필요 class·relation·attribute·axiom 정의 정도) vs **population completeness**(개별 instance·assertion·값 정도). 예: labor market ontology에 Profession class 없으면 schema 불완전, profession 일부만 있으면 population 불완전.
- 불완전 원인: 규모·복잡성(종 8.7M), 자동 IE 부정확, 적절한 source 부재, vagueness, **domain volatility**(완성 시점엔 이미 무효 — semantic change, Ch14).
- 측정: gold standard 필요하나 매우 희귀(특히 population) → **partial gold standard**(부분집합, 불완전성 드러냄 — Färber et al., Textkernel의 ESCO)나 **silver standard**(부정확하나 유용한 부분집합 — DBpedia가 2.7M typing 누락 추정). reasoning·heuristic(cardinality 위반, 평균 attribute 값 이탈 — 영화에 배우 1~2명은 드묾).
- **context-dependent**(독일 주식 목록은 독일 투자자엔 완전, 유럽 투자자엔 불완전). **주의: bias로 인한 부정확·불완전**(기여자·gold standard의 편향, Ch8).

---

## 3. Consistency
> 모델이 논리적·의미적 모순에서 자유로운 정도다.

- 예: "John의 친모는 Jane" + "친모는 Kim"(다른 사람)은 모순, disjoint class 둘 다의 instance도 모순.
- 원인: 위반 시 경고할 적절한 constraint 부재·미강제. 게으름·바쁨, framework 미지원(Neo4j는 relation cardinality constraint 미지원 → custom 필요), 또는 enforcement가 계산상 너무 복잡(OWL2 일부 profile은 consistency check가 undecidable/NP-Hard).
- **consistent ≠ accurate, inaccurate ≠ inconsistent**: 모순 없어도 부정확할 수 있음(두 비vague statement가 모순이면 둘 다 참은 불가하나 둘 다 거짓은 가능). vague statement 모순은 대개 borderline case로 inconsistent 아님.

---

## 4. Conciseness
> 모델에 redundant element가 없는 정도다.

- 예: DBpedia의 dbo:child vs dbp:children(차이 없음), Organizational Ontology의 org:memberOf vs Membership class.
- 원인: governance 부족한 다자 모델링(같은 문제에 다른 결정), 여러 app 동시 최적화(한 app엔 필요, 다른 app엔 redundant — NLP는 lexicalization 많이 필요, navigation엔 불필요), 제거 안 된 "temporary" hack·duplicate, 제거 안 된 legacy element.
- 위험: 유지 overhead·inconsistency 위험 증가, 정보가 duplicate element에 분산돼 일부만 얻을 위험, 무관 정보가 app 성능 저하(Ch10).
- 탐지: 자연어 질문을 두 가지 동등 query로 표현 가능한지(class AsianCountry vs isLocatedIn Asia), name·attribute·relation 기반 similarity로 duplicate 탐지, gold standard·corpus 대비 dead weight 점검(단 reference가 훨씬 완전하고 model이 풍부한 lexicalization 가질 때만 — verbose lexicalization은 corpus에서 발견 안 될 수 있음).

---

## 5. Timeliness
> 모델이 현재 버전의 세계를 반영하는 정도다.

- 예: Yugoslavia를 단일 국가로 여기고 후속 국가를 모르는 모델은 not timely.
- 변화 감지·반영 필요(분리된 국가 추가, 옛 국가 제거 또는 FormerCountry로). domain dynamics와 maintainer 효율에 의존(Astana→Nur-Sultan 개명이 24시간 내 Wikipedia·DBpedia Live 반영).
- 평가: 현대 지식 대비 accuracy·completeness, 또는 update 빈도·양 × domain volatility(단 error fix·과거 지식 완성이 아닌 현대 지식 관련 update여야).

---

## 6. Relevancy
> 모델의 구조·내용이 특정 task/application에 유용·중요한 정도다(application-dependent).

- 저relevancy 경우: ① 도메인 정보는 있으나 task에 critical한 정보 누락(ESCO의 entity당 synonym이 부족해 recall 낮음), ② task 관련 정보가 있으나 접근 어려움(DBpedia에 profession·skill entity는 있으나 관계 없고 Profession/Skill class로 typed 안 됨).
- 주원인: task 요구사항을 고려하지 않고 개발 → 무관할 뿐 아니라 harmful할 수 있음(Ch10).

---

## 7. Understandability
> human이 오해·의심 없이 model element를 이해·활용하는 용이성이다.

- modeler가 가장 자주 과소평가하는 dimension(계산적 속성에 치중) → 잘못된 해석·사용, accuracy·relevancy·trustworthiness 저하.
- 저understandability 원인: 나쁜 description — 모호·부정확한 이름, 난해한 axiom, human-readable 정의 부재, 미문서화된 bias·가정(Ch6).
- 평가: 문서의 명확성·구체성·풍부함 직접 질문, 또는 실제 사용 관찰해 체계적 오류 식별(rdfs:subClassOf·owl:sameAs 오용이 잦음 — 설명 부족 신호).

---

## 8. Trustworthiness
> 사용자가 인지하는 model 품질에 대한 신뢰로, 사회적·심리적 차원이 있어 수식화하기 어렵다.

- 더 부정확한 model이 더 신뢰받을 수도 있음. 요인: ① **reputation·채택**(schema.org는 Google/Microsoft/Yahoo/Yandex 설립, 웹사이트 31% 사용), ② **formal evaluation·경험 보고**(논문·기술 보고서, DBpedia/Schema.org 평가), ③ **provenance**(전문가 중앙 편집 vs 느슨한 자원봉사 community; 자동/수동 추출; source 신뢰성 — Cyc는 전문가 전용, Wikidata는 협업, DBpedia/YAGO는 Wikipedia 추출하나 community 참여 방식 다름).
- bias·creator 이익 반영 의심 시 신뢰 상실. **과장·misrepresentation은 신뢰에 해로움**(60% 주장하고 60% 달성하는 모델이 90% 주장하고 75% 달성하는 모델보다 신뢰). 제3자 model은 사용 전 scrutinize(Ch8).

---

## 9. Availability, Versatility, and Performance
> 추가 dimension — 가용성·접근 형태 다양성·효율·확장성이다.

- **availability**(존재·획득·사용 준비), **versatility**(접근 방식·형태 — DBpedia는 SPARQL·RDF 파일, ESCO는 API·RDF), **performance**(query·reasoning 효율·확장성 — framework·기술 스택 의존, 일부 OWL reasoning은 non-scalable). 본서는 이 셋보다 content·structure의 pitfall·dilemma에 집중.

---

## Summary (핵심 정리)
- 품질은 application-centered·application-neutral로 측정하며, 본서는 주로 후자를 다룬다.
- vagueness·subjectivity 하에선 statement 진위 합의가 보장되지 않으므로 accuracy 측정 시 유의한다.
- completeness는 moving target이라 정확 측정이 어렵고 partial/silver standard·heuristic을 쓴다.
- inference는 부정확을 증식·전파하며, consistent ≠ accurate, inaccurate ≠ inconsistent다.
- relevancy(채택 성패), understandability(과소평가 금지), trustworthiness(사회·심리적 차원)에 항상 주의한다.
