# 10. Distributed Data Modeling

## 챕터 개요 (3줄 요약)

- 데이터 제품 중심의 모듈·분산 아키텍처에서 데이터 모델의 정의와 모델링 프로세스(개념·논리·물리)를 설명한다.
- 분산 환경에 적용 가능한 물리 모델링 기법(차원·데이터 볼트·USS)을 다룬다.
- 분산 개념 모델링(통제 어휘·온톨로지)의 정의와 생애주기 관리를 제시한다.

---

## 1. Introducing data modeling

> 데이터 제품에서 모델은 솔루션의 일부이며, 명시적이고 공유된 개념·물리 모델 설계가 필수적이다.

- 모델은 특정 용도를 위해 일부를 강조하고 나머지는 무시하는 추상화이며, 좋은 모델은 가장 단순한 것이다.
- 사용성(usability)과 재사용성(reusability)은 상충하므로 모델러가 균형을 잡아야 한다.
- 암묵적(implicit) 모델은 머릿속에만 있어 소통 시 손실되므로, 공유된 명시적(explicit) 모델이 필요하다.
- 모델링 프로세스 3단계: 개념 모델링(정렬 alignment)→논리 모델링(정제 refine)→물리 모델링(설계 design).
- 표현 방식: 형식적(formal, 기계 판독 가능, SQL DDL/JSON Schema)과 비형식적(informal, 개념/관계 다이어그램·용어집).

### Operational vs analytical modeling
- 운영 모델링은 쓰기 최적화, 분석 모델링은 읽기 최적화이며 차이는 물리 모델 수준에서만 드러난다.
- 순수 데이터 제품은 다중 모드(multimodal)로 하나의 개념 모델에서 여러 물리 모델을 출력 포트로 노출한다.

```
   Conceptual model (align: what)
        |
   Logical model (refine)
        |
   Physical model (design: how) --> operational / analytical
```

---

## 2. Exploring distributed physical modeling

> 분석용 물리 모델링 기법(차원·데이터 볼트·USS)을 모듈·분산 환경에 맞게 조정한다.

- 차원 모델링(Kimball): 팩트(fact)와 디멘션(dimension)을 스타 스키마(star schema)/스노우플레이크로 구성한다.
- 중앙집중: Kimball(스테이징+프레젠테이션)과 Inmon(EDW Enterprise Data Warehouse 추가 3계층).
- 분산 차원 모델링: 데이터 마트를 도메인별 데이터 제품으로 관리하나, 정합 디멘션/팩트(conformed)의 소유권 귀속 문제가 발생한다.
- 데이터 볼트(Data Vault) 모델링: Hub(비즈니스 키), Link(관계), Satellite(서술 속성)로 구성한다.
- 분산 데이터 볼트: Satellite는 소스 정렬 제품이, Hub/Link는 플랫폼이 서비스로 관리해 소유권 분산을 해결한다.
- Raw Data Vault(원시)와 Business Data Vault(비즈니스 로직 적용)로 나뉜다.

### Unified Star Schema (USS)
- 조인 트랩(join trap): 팬 트랩(fan trap, 1:N 관계 측정값 중복), 캐즘 트랩(chasm trap, 공유 디멘션 교차).
- 푸피니 브릿지(Puppini Bridge): 여러 테이블을 합집합(union)으로 단일 팩트 테이블에 통합해 조인 트랩을 회피한다.
- 분산 USS: 데이터 볼트의 Hub/Link/Satellite에서 푸피니 브릿지를 플랫폼이 온디맨드 셀프서비스로 생성한다.

### Managing physical model lifecycle
- 데이터 모델은 데이터 제품의 일부로 디스크립터에 내장되어 발견성 포트로 노출되며 생애주기를 공유한다.
- 연합 거버넌스 팀이 표준을 정하고 XOps 플랫폼이 배포 시 계산적으로 검증·차단한다.

---

## 3. Exploring distributed conceptual modeling

> 데이터 제품 중심 아키텍처에서 개념 모델은 소비와 상호운용성의 핵심 enabler이다.

- 유비쿼터스 언어(ubiquitous language): 바운디드 컨텍스트의 핵심 용어를 정의하고 선호 용어를 선택한다.
- 통제 어휘(controlled vocabulary)는 KOS(Knowledge Organization Systems)로 분류되며 물리 모델 태깅에 쓰인다.
- 통제 어휘는 데이터 제품과 별개로 XOps 플랫폼/거버넌스 도구(비즈니스 용어집 business glossary)가 관리한다.

### From strings to things
- 개념·속성·관계를 구조화한 것이 온톨로지(ontology)이며 가장 표현력 높은 지식 표현이다.
- 표현 계층: 통제 어휘 ⊂ 분류체계(taxonomy, 계층) ⊂ 시소러스(thesaurus, 계층+연관) ⊂ 온톨로지.
- 온톨로지는 그래프 구조로 추론(inference)이 가능해 명시되지 않은 정보를 유추하거나 품질 이슈를 탐지한다.

### Managing conceptual model lifecycle
- 온톨로지는 실제 비즈니스 케이스에 정렬되어야 하며 완벽함이 아닌 간결·기능적 모델을 목표로 한다.
- 통합 단계에서 RDF(Resource Description Framework) 표준(subject-predicate-object 트리플)으로 형식화한다.
- 직렬화는 Turtle 포맷(간결·가독성)을 사용하며 트리플스토어(triplestore)/그래프 DB에 저장한다.
- 데이터 제품은 온톨로지 개념에 직접 링크하고 발견성 포트로 노출하며, 플랫폼이 이를 카탈로그화한다.

---

## Summary (핵심 정리)

- 데이터 제품에서 물리 모델은 소비 게이트웨이, 개념 모델은 의미 이해 프레임워크로 모델링이 제품의 일부가 된다.
- 2계층 아키텍처 제안: 소스 정렬은 데이터 볼트, 소비자 정렬은 스타 스키마/USS로 모델링한다.
- 통제 어휘에서 온톨로지까지 개념 모델링 기법을 다뤘으며, 이는 상호운용성과 AI 효과성을 높인다.
