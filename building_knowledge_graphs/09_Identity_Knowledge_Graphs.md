# 9. Identity Knowledge Graphs

## 챕터 개요 (3줄 요약)
- 정체성 지식그래프는 엔터티 해소(Entity Resolution)/마스터 데이터 관리(MDM) 문제 — "두 레코드가 같은 실세계 개체인가?"를 그래프 위상으로 해결한다.
- 핵심은 강한 식별자(SAME_AS)와 약한 식별자(가중 SIMILAR) 관계를 규칙으로 생성한 뒤, WCC(Weakly Connected Components) 알고리즘으로 연결 요소를 묶어 "골든 레코드(master entity)"를 만드는 것이다.
- 강한 식별자가 없으면 비정형 텍스트를 토큰화해 이분 그래프(product–word)를 만들고 Node Similarity(Jaccard/Overlap)로 구조적 유사도를 계산한다.

## Knowing Your Customer / When the Problem Appears
> 일관된 강한 식별자가 없으면(혼인 후 개명, 서로 다른 신분증 사용 등) 동일인 판별이 어렵다.

- 강한 식별자(주민번호·여권·SKU·법인번호)가 일관되면 쉽지만, 시스템마다 다르게 수집되면 불일치 발생.
- 중복 발생 시나리오: (1) 데이터 통합(CRM/마케팅/세일즈 각자 고객 보유), (2) 익명 활동(쿠키·클릭스트림), (3) 의도적/사기성 중복(보험 견적 조작, 무료체험 악용, 리스팅 중복).

## Graph-Based Entity Resolution Step by Step
> 엔터티 해소 3단계: 데이터 준비 → 엔터티 매칭 → 마스터 엔터티 영속화.

- 데이터 준비: 그래프로 모델링(Person 노드 + source 속성), 정규화(m_yob, m_fullname 같은 매칭용 공통 표준 속성 생성), null·대소문자·공백 정리.
- Blocking key(선택): n² 비교를 줄이기 위해 같은 블로킹 키(예: 우편번호)끼리만 비교 — 자연 키가 없으면 알고리즘/ML로 합성.
- 매칭 5단계 규칙: (1) 강한 식별자 일치 → SAME_AS, (2) 약한 식별자 임계 초과 → 가중 SIMILAR(Jaro-Winkler 등), (3) 강한 식별자 불일치 시 SIMILAR 제거, (4) 비식별 피처(생년) 일치로 점수 보정(±10%), (5) 최소 점수 미만 SIMILAR 제거.

```
// 강한 식별자 매칭
MATCH (p1:Person),(p2:Person)
WHERE p1.source<>p2.source AND (p1.ssn=p2.ssn OR p1.passport_no=p2.passport_no)
  AND id(p1)>id(p2)
CREATE (p1)-[:SAME_AS]->(p2)
// WCC로 골든 레코드 묶기
CALL gds.wcc.stream('identity-wcc') YIELD nodeId, componentId  // componentId = golden_id
```

## Build Master Entities (WCC)
> SAME_AS/SIMILAR로 연결된 노드 집합(connected component)이 곧 하나의 고유 엔터티 — WCC가 이를 탐지(무방향이라 대칭 관계에 적합).

- 프로젝션 생성 후 WCC 실행 → 각 component에 golden_id 부여 → PersonMaster 노드 MERGE + HAS_REFERENCE로 원본 연결(추적성).
- 마스터 엔터티를 소스별 상세와 함께 JSON으로 직렬화 가능. 데이터는 살아있어 델타(추가/삭제) 반복 처리 필요 — 삭제 원소 선제거 후 재매칭.
- 그래프의 차수·중심성 등 구조적 메트릭을 매칭 피처로 "자동" 활용 가능한 것이 그래프의 강점.

## Working with Unstructured Data
> 강한 식별자가 없으면 텍스트를 토큰화해 (Product)-[:includes]->(Word) 이분 그래프를 만들고 Node Similarity로 이웃 공유도를 계산한다.

- Amazon-Google 상품 매칭 사례: 제품명을 소문자·비영숫자 제거 후 split → Word 노드 연결.
- Node Similarity(기본 Jaccard, OVERLAP 옵션) + similarityCutoff로 임계 매칭 — 포맷 변화에 취약한 문자열 유사도보다 토큰 기반이 강건.
- Meredith 사례: 30B 노드/35B 관계 규모 ID 그래프에 WCC 적용 → 350M "고유" 프로필을 163M 풍부한 프로필로 통합, 고객 이해 20~30% 향상.

> [모델링 관점 - 주식시장 도메인 적용]
> 엔터티 해소는 주식시장 지식그래프의 "데이터 품질 토대"로 가장 실무적인 장이다. 금융 데이터는 동일 기업이 소스마다 다른 식별자로 등장한다(ticker vs ISIN vs CIK vs LEI vs 한글/영문 상호, 지주사/자회사 혼동). 적용: (1) 강한 식별자(LEI·ISIN·사업자번호)로 SAME_AS, 약한 식별자(상호·주소 유사도)로 SIMILAR를 만들고 WCC로 "기업 골든 레코드" 구축 — 이것이 모든 분석의 전제. (2) 비정형(뉴스·공시 본문 속 기업명 표기 변형)은 토큰/임베딩 + Node Similarity로 엔터티 링킹(named entity linking). (3) 사기/이상 탐지: 차명계좌·연계 거래자처럼 의도적 중복을 약한 식별자(공유 주소·전화·디바이스) 그래프로 묶어 적발. LEI 같은 글로벌 표준 식별자를 강한 식별자로 우선 채택하는 것이 금융 도메인 모델링의 핵심 권장사항이다.

## Summary (핵심 정리)
- 정체성 그래프는 강한 식별자(SAME_AS)+약한 식별자(SIMILAR)를 규칙으로 연결하고 WCC로 골든 레코드를 만들어 "약한 식별자들을 강한 식별자로 집약"한다.
- 강한 식별자가 없으면 토큰화 이분 그래프 + Node Similarity로 구조적 유사도를 계산하며, 이는 문자열 유사도보다 포맷 변화에 강건하다.
- 주식시장에서는 LEI/ISIN 등 표준 식별자 기반 엔터티 해소가 모든 인사이트의 전제이며, 동시에 차명·연계 거래 사기 탐지의 강력한 도구가 된다.
