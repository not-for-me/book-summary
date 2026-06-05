# 04. Identifying Data Products and Prioritizing Developments

## 챕터 개요 (3줄 요약)

- DDD(Domain-Driven Design)로 비즈니스 도메인을 모델링해 관련 비즈니스 케이스(business case)를 식별하는 방법을 설명한다.
- 이벤트 스토밍(event storming)으로 비즈니스 케이스를 지원할 데이터 제품을 발견하는 과정을 다룬다.
- 데이터 제품 포트폴리오의 검증·기술(canvas)·최적화 등 관리 활동을 제시한다.

---

## 1. Modeling a business domain

> DDD를 사용해 문제 공간(도메인)과 해결 공간(바운디드 컨텍스트)을 구조화하여 비즈니스 케이스를 분류한다.

- DDD는 도메인 전문가와 개발자가 공유 모델을 만들고 유비쿼터스 언어(ubiquitous language)로 소통하는 방법론이다(Eric Evans).
- 문제 공간=비즈니스 도메인이며 하위 도메인(subdomain)으로 계층 분해하여 모델링 가능한 단위로 나눈다.
- 바운디드 컨텍스트(bounded context): 용어·개념·규칙이 명확하고 일관된 의미를 갖는 경계로, 컨텍스트마다 별도의 유비쿼터스 언어가 존재한다.
- 하위 도메인은 문제 공간 구조화, 바운디드 컨텍스트는 해결 공간 구조화이며 가능하면 1:1로 일치시킨다.
- 1:1이 깨지면 모놀리스 또는 분산 모놀리스(distributed monolith) 위험이 생긴다.

### Identifying subdomains & bounded contexts
- 비즈니스 아키텍처(TOGAF, The Open Group Architecture Framework)에서 비즈니스 모델→가치 사슬→가치 흐름→역량 순으로 분석한다.
- 가치 사슬(value chain, Michael Porter)은 주요 기능(primary)과 지원 기능(support)으로 나뉜다.
- 비즈니스 역량(business capability)으로 하위 도메인을, 역량 인스턴스(capability instance)로 바운디드 컨텍스트를 매핑한다.
- 핵심 비즈니스 엔티티(고객·제품·주문)로 컨텍스트를 나누면 조직 구조와 직교해 관리가 복잡해진다.

### Mapping business capabilities
- 톱다운(비즈니스 모델 기반)과 보텀업(조직 구조 기반) 접근이 있으며, 대각선 전략(핵심은 빅뱅 톱다운, 세부는 기회 기반 보텀업)이 최적이다.

---

## 2. Discovering data products with event storming

> 데이터 제품은 재고 기반(stock-to-order)이 아니라 명확한 비즈니스 케이스에 근거해 이벤트 스토밍으로 발견한다.

- 전략 계획: 비전(System 5)→목표(System 4)→이니셔티브(System 3)→비즈니스 케이스(System 1) 순으로 구체화된다.
- 비전 도구로 SWOT 분석, 워들리 맵(Wardley map), 포터의 5가지 힘(Five Forces), 시나리오 플래닝 등을 사용한다.
- 비즈니스 케이스는 SMART(Specific, Measurable, Actionable, Realistic, Time-bound)해야 하며 근본 문제에 연결되어야 한다.
- 이벤트 스토밍(Alberto Brandolini)은 경량·유연한 비즈니스 프로세스 분석 워크숍이다.

### Analyzing processes with event storming
- 색상별 포스트잇 사용: 이벤트(주황), 커맨드(파랑), 액터(노랑), 시스템(분홍), 정책(보라), 문제/기회(빨강).
- 예시 LuX 사례에서 배송 지연 문제 분석 후, 재고(inventory) 데이터 제품 필요성이 도출되었다.
- 재고 데이터 제품은 4개 입력 포트(매장·이커머스 판매, 창고·진열 수량) + ERP(Enterprise Resource Planning) 검증본 입력 포트, 1개 출력 포트로 정의된다.
- 소스에서 직접 읽는 소비자 정렬 제품의 한계를 보완하기 위해 별도의 소스 정렬(source-aligned) 제품을 추가한다.

```
  POS (store sales) --+
  e-commerce sales ---+--> [ Inventory Data Product ] --> updated inventory
  warehouse qty (ERP)-+        (consolidate 5 inputs)
  shelf qty ----------+
  ERP consolidated ---+
```

---

## 3. Managing the data product portfolio

> 정의된 데이터 제품은 포트폴리오에 편입되며, 가치 극대화와 리스크 최소화를 위해 검증·기술·최적화한다.

- 검증(validation): 도메인·소유자 명시, 명확한 가치 제안, 소스 정렬 제품은 단일 소스·단일 애그리거트 읽기, 비즈니스 로직 미적용 등을 확인한다.
- 일부 규칙은 차단(blocking)이며, 예외 편입 시 비준수 사항을 제품 설명에 기록한다.
- 데이터 제품 캔버스(data product canvas): 일반 정보·입력(기대 expectations)·비즈니스 로직·출력(약속 promises)·기타를 담는 1페이지 문서이다.
- 포트폴리오 최적화: 유사 제품 병합, 미사용 제품 폐기, 분류 변경, 공통 플랫폼 컴포넌트 추출, 약속 미충족 제품 격리(quarantine) 등을 주기적으로 평가한다.
- 재사용률이 낮으면 결합성(composability) 개선을 위한 분석을 강화한다.

---

## Summary (핵심 정리)

- 비즈니스 케이스에서 출발해 DDD로 문제·해결 공간을 모델링하고, 비즈니스 역량과 그 인스턴스로 경계를 정의했다.
- 이벤트 스토밍으로 비즈니스 프로세스를 분석해 비즈니스 케이스와 데이터 제품을 식별했다.
- 정의된 데이터 제품 포트폴리오를 검증·기술·최적화하여 가치를 극대화하고 거버넌스 규칙을 준수하도록 관리했다.
