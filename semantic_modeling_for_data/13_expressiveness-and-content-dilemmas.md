# 13. Expressiveness and Content Dilemmas

## 챕터 개요 (3줄 요약)
- model에 무엇을 넣고 뺄지의 dilemma로, expressivity와 content의 적절한 균형을 자원 낭비 없이 찾는 것이 목표다.
- lexicalization·granularity·generality·negative knowledge·multiple truth·interlinking 각각에 대한 판단 기준이 있다.
- 핵심은 도메인·data뿐 아니라 element를 효과적·효율적으로 관리·적용할 수 있는 능력이 무엇을 넣을지를 좌우한다는 것이다.

---

## 1. What Lexicalizations to Have?
> lexicalization은 의미 명료화와 텍스트 탐지를 위해 필요하나, 많을수록 ambiguity가 늘어난다.

- 무한정 추가하면 안 됨 — Tomcat(동물), NLP(Neuro-Linguistic Programming) 같은 ambiguous term이 precision 저하.
- 판단: browsing/reference면 이름+1~2개로 충분(ambiguous도 OK). semantic tagging이면 **영향 평가** — 후보 lexicalization 포함/미포함 두 버전 실행해 tagging 차이가 positive/negative인지 평가.
- 대량·빈번 시 가속법: tagging 차이의 양적 패턴만 보기(기술 기사 탐지율 15%→40% 급증이면 ambiguous), 기존 model(WordNet·DBpedia)의 알려진 ambiguity 확인, WSI(word sense induction) 적용, 휴리스틱 — 이름의 축약형인가(Excel은 "뛰어나다"도), acronym인가(매우 ambiguous), 번역어인가(recuperación de datos=복원도), 품사 다중성(watch=동사/명사), 단어 수(많을수록 덜 ambiguous).
- **controversial·차별적 lexicalization 주의**(Armenian genocide 미인정, gender-neutral 직업명 부재). 안 쓰는 lexicalization은 **버리지 말고 보관**(더 나은 disambiguation 갖추면 유용).

---

## 2. How Granular to Be?
> 매우 유사하나 다른 의미의 term들을 distinct element로 모델링할지 같게 할지의 dilemma다.

- distinction 안 하는 건 pitfall이 아닌 **선택** — 이유: distinction이 복잡·비쌈(violin vs fiddle은 연주 스타일 표현 필요), subjective(SF 하위장르 Biopunk/Nanopunk), disambiguation 어렵게(fabrication 두 의미), semantic matching 어렵게(fiddle player 구인-violin 이력서 매칭엔 relatedness 필요).
- 판단 질문: 차이가 substantial·pragmatic한가 benign·theoretical한가(software engineer vs developer는 benign), distinction 유무의 결과는?, distinction을 **지원**할 수 있나(특수 정보·source·framework·app 필요).
- model은 granularity가 불균형하기 마련 — 사용자·app이 감내 가능한 수준이면 OK, 단 문서로 명확히 전달.

---

## 3. How General to Be?
> entity의 generality/specificity 차원 — 매우 general한 entity를 model에 넣을 가치가 있는가의 dilemma다.

- Analysis, Management, Technology처럼 여러 도메인을 span하고 narrower entity로 specialize되는 매우 general한 entity. **general ≠ ambiguous**(engineer라 해도 "과학 지식으로 실용 문제 해결하는 사람"은 앎).
- 문제: generic 의미가 잊히고 lexicalization이 narrower로 쓰임("technology" → 전자/SW, "diagnosis" → 의료). 정보가치 낮고 disambiguation 어렵게 함. modeler·전문가에게도 어려움(표준 upper ontology 부재, BFO 분류 실험 일치율 51%).
- 조언: ① general entity의 정보가치·ambiguity 평가(사람들이 즉시 더 구체적인 걸 떠올리면 무가치 — model엔 두되 특정 용도에서 제외), ② **on-demand로만 추가**(Civil Engineering 위에 함부로 Engineering 추가 말 것, upper ontology에서 specialize하지 말 것).

---

## 4. How Negative to Be?
> 무엇이 참이 아닌지 명시하는 negative assertion을 얼마나 넣을지의 dilemma다.

- Wikidata는 Russell이 어디서 안 태어났는지 안 적음, DBpedia는 Data Science와 다른 entity를 안 적음(but 같아 보이나 다른 것을 명시하면 유용).
- negative는 positive가 못 주는 정보를 줌(not tall ≠ short, 캐나다 거주 ≠ 타지 미거주). 단 모든 거짓을 넣는 건 비실용, framework가 negation 미지원일 수도.
- 추가가 합당한 경우: ① 다른 element로 추론·대체 불가(born in이 1개면 birthplace만, 단 TallPerson/ShortPerson은 비complementary라 둘 다 필요), ② 어떤 element 의미의 중요 부분(Marx가 안 믿은 것), ③ 흔한 오해 반박("Earth is not flat", "data scientist ≠ data engineer"), ④ IE system 개선(오탐 거부·재학습). negative를 positive만큼 신중히 다룬다.

---

## 5. How Many Truths to Handle?
> context-dependent statement의 여러 context별 진리를 표현할지(contextualization)의 dilemma다.

- vague statement는 subjective·context-dependent(Curry 191cm는 NBA 단신·일반인 장신, $25,000은 나라별 계급 다름).
- 어려움: context가 너무 많음, 식별·표현 어려움(이사회의 평가 context?), app이 context 처리 못 함(어느 skill set?).
- contextualize 가치 있는 경우(Ch12 fuzzify와 유사): 불일치 높고 consensus 어려움, context를 cost-effective하게 식별, app이 활용 가능, 불일치 실제 감소, 이득이 관리 overhead 상회.
- 조언: **minimum viable**로 시작(대륙·국가부터, 도시·우편번호 전에; 핵심 business function부터). 복잡 context model은 reasoning 필요(Europe 참 → 모든 유럽국 참 추론). **contextualization과 fuzzification은 alternative가 아닌 complementary**.

---

## 6. How Interlinked to Be?
> 외부 model과 interlink할지의 dilemma — Berners-Lee의 Linked Data 4규칙에도 불구하고 risk-benefit 분석이 중요하다.

- 외부 model과 단순 interlink로 가치가 자동 상승하지 않으며 오히려 악화될 수 있음. interlink 전 4가지: ① 외부 model 품질·호환성 scrutinize(낮은 사전 신뢰), ② 외부 model의 관리·evolution 전략(잦은 변경이 내 model에 ambiguity 유입), ③ interlink 비용·노력(다른 context 때문에 비쌈), ④ benefit(주로 도메인 표준일 때 — 단순 content면 차라리 merge).
- **ESCO 사례**(Textkernel, 2017): 구조는 유사하나 granularity 차이(Data Scientist≠Engineer 구분 여부), lexicalization이 텍스트 탐지에 미최적화, essential/optional relation이 부정확. 그러나 ESCO는 유럽 표준·일부 언어 보유·연 1회 안정적 변경. → **중간 접근**: entity는 ESCO에 link하되 ESCO의 추가 정보(lexicalization·attribute·관련 entity) 자동 유입은 금지하고 유용·정확·호환되는 것만 선별 incorporate.

---

## Summary (핵심 정리)
- lexicalization은 ambiguity가 app·user에 문제 안 되는 한 유용 — 신중히 추가한다.
- semantic distinction은 복잡·비쌀 수 있으니 중요한 것에 집중한다.
- 매우 general한 concept은 문제될 수 있고 유용하지 않으니 신중히 쓴다.
- 강한 vagueness 하에선 truth contextualization을 고려하며, 이는 fuzzification과 직교(complementary)한다.
- negative knowledge를 경시하지 말 것 — positive보다 유용·신뢰할 수 있다.
- interlinking은 늘 이롭지 않으며, 가치가 있을 때 신중히 한다.
