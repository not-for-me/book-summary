# 02. Data Products

## 챕터 개요 (3줄 요약)

- 데이터 제품(data product)과 순수 데이터 제품(pure data product), 분석 애플리케이션(analytical application)의 개념을 구분하여 정의한다.
- 순수 데이터 제품의 핵심 특성(관련성·정확성·재사용성·결합성)과 내부 구조(데이터·메타데이터·애플리케이션·인프라·인터페이스)를 분석한다.
- 순수 데이터 제품을 역할·도메인 기준으로 분류하는 방법을 제시한다.

---

## 1. Defining a data product

> 순수 데이터 제품은 분석 애플리케이션과 달리, 데이터 그 자체를 노출하는 것을 목표로 하는 모듈화 단위이다.

- DJ Patil의 정의("데이터를 활용해 목표를 달성하도록 돕는 제품")는 명확하나 너무 광범위하여 실용성이 부족하다.
- 디지털 애플리케이션은 트랜잭션 애플리케이션(데이터=부산물)과 분석 애플리케이션(데이터=핵심)으로 나뉜다.
- 확장된 정의: "데이터의 사용이 기능 개발을 지원할 뿐 아니라 견인하는 제품"이 데이터 제품이다.
- 순수 데이터 제품(pure data product)은 고품질 데이터를 순수한 형태로 노출하는 것이 목표이며, 데이터 자체가 제품이다.
- 명시적(overt) vs 은닉적(covert) 데이터 제품: 리포트는 매우 overt, 추천 시스템은 매우 covert하다.

### Why do we need pure data products?
- 가치 측정은 선행 지표(leading)와 후행 지표(lagging)를 균형있게 결합한 성공 지표(MoS, Metrics of Success)로 해야 한다.
- ETO(Engineer To Order) 방식은 분석 애플리케이션마다 통합을 독립 수행해 사일로를 재생산한다.
- ATO(Assemble To Order) 방식이 바람직하며, 순수 데이터 제품을 미리 만들어 조립한다.
- MTS(Make To Stock)처럼 사용 사례 없이 데이터 제품을 만드는 것은 자원 낭비이며, 사용 사례가 개발 우선순위를 결정한다.

### Pure data product definition
- "순수 데이터 제품은 데이터 아키텍처 내 모듈화 단위로, 책임 팀의 인지 역량에 맞춰지고 제품 관리 원칙을 따라 데이터 자산을 정확·관련·결합·즉시 사용 가능하게 만든다."
- 핵심 요소: 아키텍처 양자(architectural quantum), 인지 적합성(cognitive fit), 제품으로서의 데이터, 유동적 데이터 자산(liquid data asset).
- Moody & Walsh의 데이터 가치 7법칙(예: 무한 공유 가능, 사용할수록 가치 증가, 소멸성 등)에 따라 데이터는 일반 자산과 다르다.

### The rise of data-driven applications
- 데이터 통합 기술(스트리밍)과 분석에서의 AI(Artificial Intelligence) 활용 증가로 데이터 가치 사슬이 선형에서 순환형(continuous intelligence)으로 변한다.
- 순수 데이터 제품은 트랜잭션·분석·데이터 주도(data-driven) 애플리케이션의 중심에 위치한다.

```
   Transactional ---> [ Pure Data Product ] ---> Analytical
        ^                                            |
        |          (actionable insights)             |
        +--------------------------------------------+
        Circular value stream (continuous intelligence)
```

---

## 2. Exploring key characteristics of pure data product

> 순수 데이터 제품의 품질은 관련성·정확성(가치 유지)과 재사용성·결합성(가치 증대)의 네 가지 "ility"로 평가한다.

- FAIR(Findability, Accessibility, Interoperability, Reusability)는 공유 측면만 다루고 식별·지속성은 다루지 않는다.
- Zhamak Dehghani(데이터 메시)는 8가지 비협상 특성을 제시한다.
- 관련성(Relevance): 비즈니스 정렬, 적시 갱신, 비중복성. 데이터는 소멸성이 있어 신선할수록 관련성이 높다.
- 정확성(Accuracy): 완전성, 일관성, 유효성, 신뢰성, 정밀성. 최소 임계치 미만이면 가치가 없다.
- 재사용성(Reusability): 발견 가능, 주소 지정 가능, 이해 가능, 접근 가능, 신뢰 가능(SLA, Service-Level Agreement), 보안(GDPR, General Data Protection Regulation 준수).
- 결합성(Composability): 기술적·구문적·의미적 상호운용성과 비교적 안정적인 데이터 계약(data contract)이 필요하다.

### 가치와 변수의 관계
- 데이터 가치는 양(volume)이 과도하면 정보 과부하(information overload)로 오히려 감소한다.
- 가치는 시간(time)이 지나면 감소, 정확성(accuracy)·사용(usage)·통합(integration)이 늘면 증가한다.

---

## 3. Dissecting the anatomy of a pure data product

> 순수 데이터 제품은 데이터, 메타데이터, 애플리케이션, 인프라, 인터페이스로 구성되어 독립적 생애주기 관리가 가능하다.

- 데이터: 변환된(enriched) 데이터만 인터페이스를 통해 노출하며(정보 은닉), 다른 제품 데이터 복사를 최소화한다.
- 메타데이터: 제품 디스크립터(data product descriptor) 문서에 기계 판독 가능 형식으로 담기며, 공개 부분이 데이터 계약(data contract)이다.
- 메타데이터 책임은 중앙 스튜어드에서 제품팀으로 이동(shift left)하며, CICD(Continuous Integration/Continuous Delivery)로 강제한다.
- 애플리케이션·인프라: 소비자에게 보이지 않으며, 공유 인프라는 멀티테넌시(multi-tenancy)로 격리되어야 한다.
- 인터페이스(포트): 입력·출력·발견성·제어·관측성 포트로 구분된다.

### Interfaces (Ports)
- 입력 포트(input)는 데이터 획득 방식, 출력 포트(output)는 노출 데이터·API를 명세한다.
- 발견성(discoverability)·제어(control)·관측성(observability) 포트는 중앙 표준화된 API로 정의한다.
- API 명세는 OpenAPI, gRPC, AsyncAPI, GraphQL 등을 사용하며, 약속(promises: SLA/SLO)과 기대(expectations)를 정의한다.

```
            +-------------------------------+
  input --> |        Pure Data Product      | --> output
            |  (hexagonal: internal logic   |
  control-->|   hidden behind ports)        |--> observability
            +-------------------------------+
                       |
                 discoverability
   [Inspired by Hexagonal Architecture, Alistair Cockburn 2005]
```

---

## 4. Classifying pure data products

> 순수 데이터 제품은 아키텍처 내 역할과 도메인 정렬 방식에 따라 다양하게 분류된다.

- 소스 정렬(source-aligned): 단일 소스에서 데이터를 읽어 구문적 변환만 하며 소스 의미를 유지한다.
- 집계(aggregation): 여러 제품의 데이터를 연결·정규화하나, 의미적 정규화 외에는 권장되지 않는다(데이터 가상화 계층 활용 권장).
- 소비자 정렬(consumer-aligned): 다른 제품 데이터를 읽어 새 사용 사례를 위해 enriched 데이터를 생성한다.
- 도메인 정렬(domain-aligned): 단일 도메인 내 데이터만 다루며 채택 초기에 먼저 개발된다.
- 가치 흐름 정렬(value stream-aligned): 도메인을 가로지르는 전사 프로세스(예: order to cash)를 지원하며 더 복잡하다.

### Other classifications
- 성숙도·크기·위험 수준 등으로도 분류 가능하며, 거버넌스·보안 정책에 따라 계층(tier)으로 메타 분류하기도 한다.

---

## Summary (핵심 정리)

- 본 챕터는 분석 애플리케이션과 구분되는 "순수 데이터 제품"의 실행 가능한 정의를 제시하여, 데이터 아키텍처의 모듈화 단위로 자리매김했다.
- 관련성·정확성·재사용성·결합성의 네 특성과 데이터·메타데이터·애플리케이션·인프라·인터페이스의 구성 요소를 분석했다.
- 순수 데이터 제품은 배포부터 자율적 생애주기 관리까지 필요한 모든 구성 요소를 포함하며, 데이터 자산의 잠재 가치를 다중 사용 사례로 확장한다.
