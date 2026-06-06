# 14. Asking a KG Questions with Natural Language

## 챕터 개요 (3줄 요약)

- 법집행(policing) 도메인을 사례로, RAG의 한계를 넘어 도메인 전문가의 KG 질의 전문성을 모방하는 "전문가 에뮬레이션(expert emulation)" 질의응답 시스템을 구축한다.
- 핵심 독창 개념: 답 생성에서 올바른 질문(자연어→Cypher 쿼리) 만들기로의 패러다임 전환, 의도 탐지(intent detection), 메타데이터로 강화된 스키마, 전문가식 추론.
- 의도 탐지→스키마 추출→쿼리 생성→실행→시각화→요약의 파이프라인으로 비기술 전문가가 KG를 효과적으로 질의하게 한다.

---

## 1~2. Policing Domain & RAG Capabilities/Challenges

> RAG는 모든 관련 맥락이 검색될 때만 효과적이며, 검색이 불완전하면 취약하다.

- 도메인 전문가(분석가)에게 직접 KG 접근권을 주면 전문성 활용, 적시 결정, 기술 장벽 극복, ROI 극대화, 협업 등 큰 이점이 있으나, 전통적으로 KG 질의는 깊은 기술 능력을 요한다.
- 완전한 맥락의 RAG: 목격자 진술 전체를 제공하면 LLM이 "범인은 왼손잡이인가?"에 정확히 답한다(요약·분석에 강함).
- 불완전 검색의 취약성: 단 하나의 목격자 진술(Witness C)이 빠지면 모델이 정반대 결론(오른손잡이)을 내린다 — RAG 출력은 검색 단계만큼만 좋다.
- RAG는 답이 여러 문서에 걸치거나 검색이 모두 포착 못 하면 파편화된 맥락으로 그럴듯하나 부정확한 답을 낳는다.

---

## 3. Schema-Based Approach for Querying KGs

> 전문가는 그래프의 청사진인 스키마를 먼저 이해하고 자연어 질문을 정밀한 순회(traversal)로 변환한다.

- 스키마 이해는 "이 지역에서 본 빨간 카마로" 같은 요청을 "Person-Owns_car-Car-Captured_by-ANPR" 순회로 변환하게 한다.
- 자연어→Cypher 변환 3단계: 질문을 의미 요소로 파싱, 요소를 스키마 요소(노드·관계)에 매핑, 제약 조건을 적용한 정형 쿼리 구성.
- 제약 순회(constrained traversal)가 어떤 복잡도의 정형 쿼리도 구성하는 기본 빌딩 블록이다.

---

## 4. Think Like an Expert: Using Metadata

> 전문가 추론을 LLM이 수행 가능한 작업 집합으로 변환할 수 있다.

- 핵심 패러다임 전환: 답 생성이 아니라 올바른 질문(정형 쿼리) 만들기에 초점 — 쿼리는 KG 구조와 상호작용하도록 형식화된 질문이다.
- 시스템 구성: 의도 탐지, 스키마 추출, 쿼리 생성, 쿼리 실행, 시각화, 요약 생성.

---

## 5. Intent Detection: Understanding User Expectations

> 사용자 의도를 이해해 적절한 표현 방식과 파이프라인을 선택한다.

### 14.5.1 Classifying by Visualization Type

- 출력 표현 분류: graph(노드·관계 캔버스), table(집계·정렬·통계), chart(분포 플롯), map(위치·장소).
- 좋은 분류 프롬프트: 명확한 지시, 정의된 범주, 예시, 경계 사례(boundary case), 출력 형식, 폴백 옵션. graph를 catch-all 클래스로 편향시킨다.
- reason 필드는 다운스트림에 쓰지 않으나 디버깅·오분류 식별에 유용하다.

### 14.5.2 Is it Data, Documentation, or Just Complaining?

- 비기술 전문가를 위해 데이터 관련, 시스템 관련(문서 관련/스키마 관련 하위 분류), 피드백·불만의 전체 의도 스펙트럼을 다룬다.
- "KG는 얼마나 자주 갱신되나?" 같은 질문에서 대형 모델은 시스템 관련(문서)로, 소형·양자화 모델은 데이터 관련으로 오분류하며, reason 필드가 맥락 부족을 드러낸다.
- 단일 광범위 프롬프트(단순·빠른 배포) vs 다단계 분류(정확성·유연성)의 트레이드오프를 고려한다.

---

## 6. From Schema to LLM-Ready Context

> 원시 스키마 정보를 LLM이 처리 가능한 형식으로 변환한다.

### 14.6.1 Schema Extraction and Representation

- apoc.meta.schema는 기술 DB 스키마를 주나 헬퍼 노드·기술 메타데이터 등 불필요한 세부가 많아, 도메인 핵심 엔티티·관계만 담은 개념 스키마(conceptual schema)가 필요하다.
- 개념 스키마의 이점: 인간 추론과 정렬, LLM 인지 부하 감소, 쿼리 오류 최소화, 해석가능성 향상.
- LLM 친화적 형식은 엔티티명·속성·관계만 담은 간결·일관된 구조로, 장황한 서술형(narrative)보다 효율적이다.

### 14.6.2 ~ 14.6.3 Enriching Schemas with Annotations

- 스키마 구조만으론 부족: "black"이 DB에 "BLK"로 저장되거나 COMMITTED vs CO_OFFENDS_WITH 관계 모호성 문제가 생긴다.
- 전문가의 "치트 시트"(용어·약어·관계 의미)를 모방해 노드·관계·속성에 인라인 주석(/* */)으로 스키마를 체계적으로 주석한다.
- YAML 설정 파일로 skip(무관 요소 필터링)과 descriptions(주석 추가) 섹션을 관리해 커스터마이징·유지보수성·확장성을 확보한다.

```
  (:Vehicle {
    color: STRING, /* Color of vehicle, BLK, GRY, SIL, WHI, etc */
    make: STRING   /* Manufacturer: BMW, BUIC, CADI, CHEV, etc */
  })
  (:Vehicle)-[:OWNED_BY {since: DATE}]->(:Person)
```

---

## 7. It's Time to Think: Understanding LLM Reasoning

> LLM이 결론으로 "서두르지" 않도록 추론할 시간을 줘야 한다.

- Chain-of-thought 프롬프팅(단계별 추론 유도)과 scratchpad 기법(중간 작업 토큰 생성)으로 문제 해결에 더 많은 연산을 할당한다.
- 순서가 중요: 답을 먼저 시키면 모델이 초기 답을 고수하고 그에 맞춘 추론을 생성하므로(의미적 일관성 semantic consistency), 추론을 먼저 시킨다.
- 누적 맥락(cumulative context)과 오류 전파(error propagation) 때문에 추론 우선이 투명·신뢰성 있는 결과를 낳는다(분류 작업은 예외적으로 reason을 뒤에 둠).

### 14.7.2 ~ 14.7.3 Thinking in Queries: Text to Cypher

- 쿼리 생성 프롬프트 구조: 작업 설명·질문, 주석 스키마 정의, 의도 의존 요구사항, 예시, 선택적 사용자 선택(selection), KG 특화 주석, 질문·요구사항 재확인, 출력 형식.
- 출력 JSON 필드 순서: relationships(순회할 관계 "소리내어" 나열로 환각 감소), reasoning(스크래치패드), query(Cypher), success.
- 관계를 먼저 나열시켜 존재하지 않는 관계 사용을 막고, 인간 전문가처럼 단계별로 진행해 신뢰성 있는 결과를 얻는다.

---

## 8. Response Summarization: From Results to Insights

> 요약은 실제 데이터에 접근하는 유일한 단계로, 원시 데이터와 사용자 이해의 간극을 메운다.

- 그래프 시각화는 관계·구조를 잘 보여주나, 노드 속성·넓은 맥락의 가치 있는 정보는 요약이 강조한다(시각화를 반복이 아닌 보완).
- 프롬프트는 질문→쿼리→결과→선택의 맥락 사슬을 HTML 유사 태그로 재구성하고, 무관 데이터 필터링을 명시 지시한다.
- "분석이 요청되었나?" 플래그로 사용자 의도에 묶인 분석을 수행하며, results_analysis·reasoning·summary를 JSON으로 점진 생성한다.

---

## Summary (핵심 정리)

- 전문가 에뮬레이션은 "전문가라면 어떻게 할까?"를 구현 가능한 단계로 분해해 KG 시스템을 구축·개선·확장하는 체계적 프레임워크다.
- 의도 탐지는 광범위 범주(데이터·시스템·피드백)와 시각화 필요(graph·table·chart·map)의 두 계층 분류를 요하고, 기술 스키마는 불필요 요소 필터링·맥락 주석을 거쳐 LLM 친화적 형식으로 변환한다.
- 프롬프트는 LLM에 "생각할 시간"(추론 우선, chain-of-thought)을 줘야 하며, 쿼리 생성은 스키마 맥락·선택 상태·의도별 요구·예시가 필요하고, 결과 요약은 시각화를 보완해 잘 드러나지 않는 통찰을 강조한다.
