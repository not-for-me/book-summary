# 13. Talking to Your Knowledge Graph

## 챕터 개요 (3줄 요약)
- 자연어와 지식그래프의 상호작용은 3가지: (1) 자연어→그래프 입력(사실 추출), (2) 그래프→자연어 출력(텍스트 생성), (3) 그래프가 NLP 작업의 구조적 맥락 제공.
- 사실 추출(fact extraction)로 질의응답 그래프를 만들고, 자연어 질의를 Cypher로 변환하는 대화형 인터페이스를 구현하며, 그래프 구조로부터 자연어를 생성한다.
- WordNet 같은 어휘 데이터베이스를 그래프로 만들면 path/Leacock-Chodorow/Wu-Palmer 의미 유사도를 투명하게(블랙박스 아님) 구현·확장할 수 있다.

## Question Answering (Fact Extraction)
> NER이 "이 문서가 무엇에 관한가"라면, 사실 추출은 "Tesla의 CEO는 누구인가"에 답하는 그래프를 만든다 — NLP 역할이 엔터티→사실로 이동.

- Diffbot 등은 entity-property-value 트리플(humanReadable, confidence, Wikidata 교차참조 포함)을 추출 → 노드 속성/관계로 모델링.
- 도전 과제: 관계 조화(harmonization) — 관계 타입 난립 방지를 위해 어떤 관계가 유의미한지 입력 필요.

## Natural Language Query → Cypher
> Cypher는 기술자만 접근 가능 — 자연어 인터페이스가 비기술 사용자에게 그래프 가치를 개방한다.

- spaCy 규칙 기반 Matcher: 패턴(LOWER/LEMMA/POS/OP 수식어)으로 "Who wrote X?" 같은 질문 구조 포착 → 동사→관계 매핑 사전으로 Cypher 생성.
- 그래프는 자기 기술적(self-describing)이므로 `db.schema.*`로 스키마를 읽어 조직 원리에 주석을 달면 인터페이스가 그래프 진화에 맞춰 자동 갱신.

## Natural Language Generation
> 그래프의 "명사=노드, 동사=관계" 구조를 역으로 읽으면 subject-predicate-object 문장을 생성할 수 있다.

- 완전 제네릭 쿼리: `(x)-[r]->(y)` 연결+속성으로 "Keanu Reeves acted in Johnny Mnemonic" 같은 영어 생성 — 그래프가 인간과 유사하게 도메인을 포착하는 경험적 증거.
- 한계: 노드 명명 방식(name 속성 가정), 스키마 의존(sub_109 같은 비가독 관계명), 역방향 탐색의 언어적 비대칭.
- 해결: 온톨로지(OWL/SKOS) 주석으로 talk:direct/talk:inverse(역방향), 길고/짧은 버전, $s/$o 파라미터를 정의 → Cypher NLG 엔진이 임의 노드를 자연어로 기술. Rasa 봇 프레임워크 연동 가능.

## Working with Lexical Databases (WordNet) & Semantic Similarity
> WordNet(synset=인지적 동의어 집합)을 RDF로 임포트하면 Form–LexicalEntry–LexicalSense–LexicalConcept 그래프가 된다.

- 다의어(polysemic) 분석(break=75 의미), 역조회(한 개념을 표현하는 모든 단어), hypernym/hyponym 택소노미 탐색.
- 의미 유사도 3종을 Cypher로 구현(NLTK와 동일 결과):
  - **Path similarity**: 최단경로 기반 `1/(1+pathLen)`.
  - **Leacock-Chodorow**: 택소노미 깊이 반영 `-log(pathLen/2*depth)`.
  - **Wu-Palmer**: 최소공통상위(LCS) 깊이 기반 `2*LCS_depth/(2*LCS_depth+depth_a+depth_b)`.
- Cypher 사용 이점: NLTK 버전 의존 없음, 도메인 용어 추가 가능(투명성/블랙박스 아님), 임의 택소노미(Wikidata SubClassOf 등)에 적용.

> [모델링 관점 - 주식시장 도메인 적용]
> 이 장의 기법은 주식시장 인사이트 인터페이스로 직결된다: (1) 사실 추출 → 공시/뉴스에서 "A사가 B사를 인수", "C사 CEO 교체" 같은 사실을 트리플로 추출해 그래프 관계로 적재(질의응답형). 단 금융은 관계 타입 표준화가 중요하므로 FIBO 관계 어휘로 harmonize. (2) NL→Cypher → 애널리스트/PM이 "삼성전자에 지분 5% 이상 보유한 기관은?"을 자연어로 물으면 Cypher로 변환 — 비기술 사용자의 지식그래프 접근성 확보(내 플랫폼의 핵심 사용자 경험). (3) NLG → 리스크/포트폴리오 분석 결과를 자연어 리포트로 자동 생성(예: Panama Papers식 조사 보고서 → 차명/UBO 조사 자동 서술). (4) 도메인 택소노미 유사도 → 산업분류(GICS) 택소노미에 Wu-Palmer를 적용해 "유사 섹터/업종" 정량화, WordNet에 금융 전문용어를 추가하듯 GICS에 신규 테마(AI반도체 등)를 동적으로 삽입해 유사도 계산. 핵심은 "온톨로지 주석"이 NL 인터페이스와 그래프 스키마를 분리해 도메인 진화에 대응하게 한다는 점이다.

## Summary (핵심 정리)
- 자연어×그래프는 사실 추출(입력)·NL→Cypher 질의·NLG(출력)의 3방향이며, 그래프의 자기 기술성이 이 모든 변환을 쉽게 만든다.
- WordNet 어휘 그래프로 path/Leacock-Chodorow/Wu-Palmer 유사도를 투명·확장 가능하게 구현하며, 임의 택소노미에 일반화된다.
- 주식시장에서는 공시 사실 추출·자연어 질의 인터페이스·자동 리포트 생성·산업분류 유사도에 적용되며, 온톨로지 주석이 인터페이스와 스키마를 분리하는 열쇠다.
