# 11. Bad Strategy and Organization

## 챕터 개요 (3줄 요약)
- 아무리 기술에 능해도 올바른 strategy와 organization 없이는 semantic modeling initiative가 실패한다.
- semantic model strategy는 goal·high-level approach·decision-making 메커니즘에 관한 것이며, myth 맹신·복잡성 과소평가·context 무시를 피해야 한다.
- 좋은 전략도 right team(semantic thinking 필수)·governance 없이는 실패한다.

---

## 1. Bad Strategy
> 두 pitfall: 전략·조직을 무시하고 기술·절차에만 집중, 또는 전략을 세우되 실행 context와 맞지 않음.

### 1.1 What Is a Semantic Model Strategy About?
> strategy는 model의 goal, 목표 달성 high-level approach, 실행 decision-making 메커니즘 세 가지에 관한 것이다.

- **Strategic goals**: "왜 만드나? 무엇을 강화? 누가 왜 쓰나? 어떤 가치? 안 만들면?" — 가능한 **구체적**으로(buzzword 금지: "semantic search 만들기" X, "특정 subdomain의 precision 향상" O). goal은 approach(deep learning으로 KG)가 아님.
- **High-level approach**: 상세 계획이 아닌 philosophy·원칙·우선순위. 예: ESCO(open standard RDF/SKOS, expert만, 드문 release) vs Diffbot(자체 query language, web crawling, 4~5일마다 갱신).
- **Decision-making**: 결정 자체보다 결정의 **기준·메커니즘** 정의. **product owner의 중요성**(Textkernel 초기 product owner 부재로 작업 낭비).

### 1.2 Buying into Myths and Half-Truths
> 현실적 landscape 지식이 있어야 하며, 흔한 myth를 경계한다.

- myth들: "Semantic Web 언어로만 가능"(Cyc·LinkedIn KG는 비RDF), "데이터에서 거의 자동으로 채울 수 있음"(당신 요구가 그들과 같은가?), "공개 model 쉽게 재사용"(부정확·부적합), "semantic interoperability=같은 언어"(실은 의미 합의 필요), "uncertainty·vagueness는 noise"(위험한 조언), "논문 X의 방법 그대로 쓰면 됨"(재현 어려움 — 건전한 회의로 접근).

### 1.3 Underestimating Complexity and Cost
> 전체 노력의 복잡성을 과소평가하는 흔한 문제(reference data 부족).

- "How Much Is A Triple?": 수동 triple $2~6, 자동 $0.0083~0.14, 고비용 statement가 더 정확. 비용 증가 요인: 도메인의 다양성·nuance, element 복잡성(복잡 relation·axiom), entity의 abstractness, vagueness 강도, 적용·재사용 범위, data source의 구조·명시성, infra·기술·process 성숙도.

### 1.4 Not Knowing or Applying Your Context
> 다른 model·조직이 한 것에 기반하지 말고 자신의 context를 조사·적용한다.

- "model X가 같은 도메인 cover한다고 모든 relation이 필요한가? 회사 Y가 자동 개발했다고 그 품질이 우리 기준에 맞나?" — 도메인·조직·고객·사람·경쟁자·기술을 **넓고 깊게** 이해. context를 안다 = 강약점을 알고 전략적 결정(수익 큰 product 최적화, vague하면 multiple truth, 50% 추출 시 수동 curation).
- **Neo4j 사례**: Textkernel이 Neo4j 사용 — Semantic Web 사람들은 RDF 권하나, 기존 Neo4j 작업을 버릴 business case가 있어야. context도 변하므로 전략은 일회성이 아닌 지속적 form·apply·monitor·revise.

---

## 2. Bad Organization
> 좋은 전략도 잘 실행되지 않으면 실패하며, right people·skill·process가 필요하다.

### 2.1 Not Building the Right Team
> semantic data modeling은 data만큼이나 semantics에 관한 것이라 어느 한쪽도 소홀히 하면 실패한다.

- ontologist만이면 너무 느리고, logician만이면 real-world data(ambiguity·vagueness)에 안 맞고, data scientist만이면 원치 않는 semantics 추출.
- **필요 skill**: 근본은 **conceptual·semantic thinking**(특정 언어 지식보다 어떤 framework든 이해·올바른 사용) — semantic phenomena 식별(ambiguity vs vagueness), element 올바른 사용(subclass로 part-whole 표현 안 함, domain/range는 inference rule), 사람들의 terminology 해독, 사용자 입장에서 오해·bias 예상. 그 외 IE/NLP/ML, data engineering, UX/UI, domain expertise, model 적용 경험(librarian·taxonomist).
- **피해야 할 attitude**: pedantic(불필요한 이론적 distinction 집착), semantics nihilist(명백한 distinction 무시), 모든 게 boolean이라 믿음, data/expert/crowd 광신, "망치 있으니 모든 게 못"(문제를 해법에 맞춤). 최적 균형이 관건.

### 2.2 Underestimating the Need for Governance
> 결정 메커니즘은 조직적 이슈로, 명확한 규칙·역할·decision right·process·accountability 체계가 필요하다.

- laissez-faire의 위험: semantic divergence·자원 낭비.
- **semantic divergence story**: Textkernel의 두 팀이 같은 KG 부분을 다른 client용으로 독립 확장 → granularity(synonymy 태도) 차이로 merge가 어려워 2~3개월 추가 소요(처음부터 협업했으면 회피 가능).
- **governance framework 질문**: 어떤 변경이 어떤 상황에서 허용? 누가 변경 가능?(전문가 제한) 어떻게 변경?(승인·테스트 process) 누가 소유·책임? 누가 결정·갈등 해결?(Wikipedia consensus) 원칙·결정을 어떻게 강제?(FIBO의 community standard 위반 영구 ban).

---

## Summary (핵심 정리)
- strategy엔 goal·high-level approach·decision-making 메커니즘을 반드시 포함한다.
- goal은 "왜 만드나? 안 만들면?"으로 정의한다.
- semantic model 구축엔 vendor가 말하는 것보다 많은 기술·framework 옵션이 있다.
- semantic interoperability는 같은 언어가 아닌 의미 합의의 문제다.
- 비용은 도메인 다양성·entity의 abstractness·vagueness·재사용 범위에 비례한다.
- semantic modeling은 data만큼 semantics에 관한 것이며, team이 semantically 사고할 수 있어야 한다.
