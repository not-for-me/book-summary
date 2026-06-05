# 11. Building an AI-Ready Information Architecture

## 챕터 개요 (3줄 요약)

- 데이터 제품을 기반으로 정보 아키텍처의 계층(데이터·정보·지식·지능)을 설명한다.
- 엔터프라이즈 온톨로지(enterprise ontology)와 지식 메시(knowledge mesh)로 지식 평면을 관리하는 방법을 다룬다.
- 엔터프라이즈 지식 그래프(knowledge graph)를 구축해 현대 생성형 AI(Artificial Intelligence)를 활용하는 법을 제시한다.

---

## 1. Exploring information architecture

> 데이터 자산의 가치를 극대화하려면 데이터 계층뿐 아니라 정보 아키텍처 전체를 관리해야 한다.

- 사용 가치(value in use)가 있는 데이터만 자산이며, 그렇지 않으면 부채(liability)이다.
- 정보 아키텍처 피라미드: 데이터(raw)→정보(information, 메타데이터로 맥락 부여)→지식(knowledge, 개념 모델)→지능/지혜(intelligence).
- 메타데이터(metadata)는 원시 데이터를 이해·사용 가능한 정보로 변환한다.
- 지식은 도메인 모델로, 정보의 조직 내 깊은 의미를 파악하고 가치를 창출하게 한다.

### Information & knowledge planes
- 순수 데이터 제품은 데이터와 메타데이터(데이터 평면+정보 평면)를 모두 관리하며, 개념 모델은 별개로 참조만 한다.
- 서브도메인 온톨로지(subdomain ontology)는 연합 모델링 팀이 독립 개발하나 서로 연결되지 않아 의미적 격차(semantic gap)가 생긴다.
- 도메인 온톨로지(domain ontology)는 공통 핵심 개념을 정의해 서브도메인 간 의미적 상호운용성을 보장한다.
- 지식 모델링을 미루면 단기 절감되나 소비 시 비용이 배가된다(선제적 관리 필요).

```
        Intelligence / Wisdom  (act, decide)
        Knowledge  (ontology / domain model)
        Information  (data + metadata)
        Data  (raw facts)
   [Information Architecture Pyramid]
```

---

## 2. Managing enterprise knowledge

> 엔터프라이즈 온톨로지로 지식을 모델링하고 연합 모델링 팀이 제품처럼 관리한다.

- 엔터프라이즈 온톨로지는 추상화 수준별 계층 구조이다.
- 상위 온톨로지(upper ontology): object·person·time 등 보편 개념(표준: BFO Basic Formal Ontology, GIST).
- 도메인 온톨로지: 상위 도메인(조직 간 공유)과 핵심 도메인(특정 조직)으로 세분화 가능하다.
- 서브도메인 온톨로지: 필요 시에만 확장하며, 애플리케이션 온톨로지로 BI 뷰를 기술한다.
- 지식 도메인(knowledge domain): 비즈니스 엔티티·프로세스 기준으로 조직을 분류하며 서브도메인과 직교한다.

### Federated modeling team & knowledge mesh
- 연합 모델링 팀은 각 서브도메인 대표 + 모델링 전문가(정보 아키텍트·데이터 스튜어드)로 구성된다.
- 온톨로지는 비즈니스 필요에 의해 주도되며 제품처럼(스트림 정렬 팀이 XaaS로 제공) 관리한다.
- 지식 메시(knowledge mesh): 데이터 메시의 4원칙을 지식 평면에 적용한 접근법이다(KnowledgeOps).

---

## 3. Building an enterprise knowledge graph

> 엔터프라이즈 온톨로지를 데이터 제품과 연결해 정보 아키텍처 전체를 단일 모델로 표현한다.

- 지식 그래프 아키텍처: 구체화형(데이터 중심 data-centric, 지식 웨어하우스)과 논리형(가상 virtual knowledge graph).
- 논리 지식 웨어하우스를 권장(참조 데이터를 언제든 구체화 가능하나 역은 불가).
- DPDS(Data Product Descriptor Specification)의 s-context 어노테이션으로 데이터와 온톨로지 개념을 의미적으로 연결한다.
- 표준 온톨로지 사용: DPROD(데이터 제품), ODRL(사용 정책), PROV-O(계보), DQV(품질), R2RML/RML(매핑).
- 엔터프라이즈 지식 그래프 = 메타데이터 온톨로지 + 매핑 온톨로지 + 도메인 온톨로지 + 인스턴스.
- SPARQL 쿼리·RDF(Resource Description Framework) 표준으로 통합 검색 경험을 제공한다.

---

## 4. Leveraging modern AI

> 견고한 정보 아키텍처는 생성형 AI의 한계를 보완하고 더 많은 비즈니스 케이스를 가능케 한다.

- 생성형 AI(generative AI)의 3대 과제: 전문 지식 부족, 예측 불가능한 출력, 제한된 추론 능력.
- RAG(Retrieval-Augmented Generation): 조직 데이터를 검색해 모델에 맥락으로 전달하나 모델이 개념을 직접 추론해야 한다.
- GraphRAG: 엔터프라이즈 온톨로지의 개념·관계까지 전달해 추론을 강화하고 정확도를 높인다.
- 뉴로-심볼릭 AI(neuro-symbolic AI): 신경망(통계)과 기호 AI(지식 그래프)를 결합하며, Kahneman의 System 1(빠름)·System 2(분석적)에서 영감을 얻는다.
- 미래 AI도 조직 고유 지식을 어떤 형태로든 습득해야 하므로, 도메인 지식 형식화 투자가 미래 대비책이다.

```
   neural networks (System 1: fast, intuitive)
            +  
   symbolic AI / knowledge graph (System 2: slow, logical)
            =  neuro-symbolic AI
```

---

## Summary (핵심 정리)

- 데이터는 메타데이터로 정보가 되고, 정보는 상호 연결된 개념으로 명시적 지식 모델이 된다.
- 도메인 온톨로지는 조직 공통으로 지식 사일로를 방지하고 의미적 상호운용성을 보장하며 연합 모델링 팀이 점진 관리한다.
- 엔터프라이즈 지식 그래프는 데이터·정보·지식을 단일 모델로 통합해 사람과 AI 에이전트가 도메인 비즈니스 케이스를 고정확도로 수행하게 한다.
