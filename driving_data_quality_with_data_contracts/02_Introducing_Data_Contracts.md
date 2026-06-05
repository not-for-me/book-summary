# 02. Introducing Data Contracts

## 챕터 개요 (3줄 요약)

- 데이터 컨트랙트(data contract)는 "데이터를 위한 API"로, 생성자와 소비자 간 합의된 인터페이스(agreed interface)다.
- 인터페이스, 기대치 설정, 거버넌스 정의, 품질 데이터의 명시적 생성이라는 네 가지 핵심 원칙으로 구성된다.
- 데이터 메시(data mesh)와 목표가 일치하며, 데이터 컨트랙트는 메시를 구현하는 데 필요한 인터페이스와 툴링을 제공한다.

---

## 1. What Is a Data Contract?

> 데이터 컨트랙트는 생성자와 소비자가 합의한 인터페이스로, 데이터에 대한 기대치와 거버넌스를 정의하고 품질 데이터의 명시적 생성을 가능케 한다.

- 정의: 데이터 생성자와 소비자 간 '합의된 인터페이스'로, 기대치를 설정하고 거버넌스 방식을 정의하며 품질 데이터의 명시적 생성을 촉진한다.
- 네 가지 핵심 원칙: ① 합의된 인터페이스 ② 기대치 설정 ③ 거버넌스 정의 ④ 품질 데이터의 명시적 생성.
- 주 대상은 조직이 내부적으로 생성·통제 가능한 데이터지만, Salesforce 같은 서드파티 데이터에도 원칙 적용이 가능하다.
- API를 생성자/소비자 간 계약으로 보는 관점에서 'data contract'라는 이름이 유래했다.

### An agreed interface between generators and consumers

- 인터페이스는 소프트웨어 라이브러리·API·SOA(Service-Oriented Architecture)·산업 표준(예: ISO 8583)처럼 어디에나 존재한다.
- 인터페이스는 구현 세부를 숨기는 추상화(abstraction)를 제공해 제공자가 자율적으로 내부를 변경할 수 있게 한다.
- 인터페이스 정의 과정 자체가 생성자와 소비자를 가까워지게 하고, 생성자에게 주인의식을 부여한다.

### Setting expectations around that data

- 컨트랙트는 스키마/구조, 유효·무효 값(데이터 품질 체크), 성능·신뢰성(SLO), 소유권·책임에 대한 기대치를 설정한다.
- 스키마는 protobuf, Apache Thrift, Apache Avro, JSON Schema 등 IDL(Interface Definition Language)로 정의할 수 있다.
- 데이터 품질 체크: 최소/최대값, 정규식 매칭, 유니크 제약, 참조 무결성(referential integrity) 등.
- SLI(Service-Level Indicator)인 완전성(completeness)·적시성(timeliness)·가용성(availability)을 측정해 SLO(Service-Level Objective)를 설정한다.

### Ownership and responsibilities

- 데이터 생성자가 데이터에 대한 책임을 더 지는 '책임의 좌측 이동(shift-left)'을 추구한다.
- 생성자는 컨트랙트를 소유하고, 소비자 요구를 고려해 확장 가능한 방식으로 데이터를 제공할 자율성을 가진다.
- 데이터 컨트랙트는 생산자 주도(producer-driven) 계약이며, 이는 데이터 품질 책임을 생성자에게 두는 유일한 방식이다.

```
[Data Generator] --(producer-driven data contract / interface)--> [Data Consumer]
        ^                          ^                                     ^
    owns contract            schema + SLO + governance            builds with confidence
```

---

## 2. Defining How the Data Should Be Governed

> 데이터 컨트랙트는 데이터 거버넌스(표준·정책·프로세스)에 필요한 메타데이터를 담아 단일 진실 공급원(source of truth) 역할을 한다.

- 거버넌스 메타데이터: 개인정보 여부와 식별자 유형, 민감도(기밀/비밀/공개), 접근 권한, 보존 기간, 익명화 전략 등.
- 데이터에 가장 많은 맥락을 가진 생성자가 데이터 분류·라벨링을 결정하는 최적의 주체다.
- 컨트랙트를 기계 판독 가능(machine-readable) 형식으로 두면 프라이버시 툴, 데이터 카탈로그 등과 연동된다.
- 이를 통해 탈중앙화되고 더 효과적인 거버넌스 접근과 자동화가 가능해진다.

---

## 3. Facilitating the Explicit Generation of Quality Data

> 원천 서비스의 부산물로 추출된 원시(raw) 데이터 대신, 소비자 요구를 충족하도록 의도적·명시적으로 생성된 데이터로 전환한다.

- 기존 ELT(Extract, Load, Transform)/CDC(Change Data Capture) 데이터는 '소비용으로 만들어지지 않아' 불안정·고비용 파이프라인을 낳는다.
- 데이터 컨트랙트는 의도적으로 소비를 위해 생성된 데이터로 전환을 촉진한다.
- 생성자와 소비자를 가깝게 만들어 컨트랙트를 협업의 장으로 활용한다.
- 소비자는 데이터의 비즈니스 가치를 명확히 설명해 생성자의 참여를 유도해야 한다.
- 생성자를 위해 outbox, listen-to-yourself 같은 발행(publishing) 패턴과 툴링을 제공해야 한다.

---

## 4. When to Use Data Contracts

> 데이터를 조금이라도 사용하는 조직이라면 가능한 한 일찍 데이터 컨트랙트를 도입하는 것이 유리하다.

- 비즈니스 핵심 프로세스나 고객 대상 제품에 데이터를 쓸수록 접근성 좋은 품질 데이터가 필요하다.
- 사용자들이 가장 먼저 묻는 질문은 "이 데이터를 신뢰할 수 있는가?"이며, 컨트랙트가 이에 답한다.
- 핵심 모델은 product-market fit 이후 비교적 안정적이라 조기 도입이 속도를 크게 희생하지 않는다(GoCardless 사례).
- 늦게 도입할수록 기존 데이터 문화가 굳어 레거시 파이프라인 폐기·마이그레이션이 어려워진다.
- 대규모 조직에서는 가장 중요한 데이터 애플리케이션 몇 개로 시작해 효과(인시던트 감소, ETL 비용 절감 등)를 측정한다.
- 가장 기본 형태의 컨트랙트는 생성자와 소비자가 합의한 '문서' 한 장이며, 점진적으로 툴링을 고도화한다.

---

## 5. Data Contracts and the Data Mesh

> 데이터 메시(2019, Zhamak Dehghani)는 도메인 지향·탈중앙화 데이터 플랫폼 패턴으로, 데이터 컨트랙트가 그 인터페이스와 툴링을 제공한다.

- 데이터 메시 네 원칙: 도메인 소유권, 데이터를 제품으로(data as a product), 셀프서브 데이터 플랫폼, 연합형 계산 거버넌스(federated computational governance).
- 도메인 소유권: 컨트랙트가 소유권과 도메인 메타데이터를 정의하는 인터페이스를 제공한다.
- 데이터 제품: 컨트랙트는 데이터 제품의 인터페이스로, 제품들이 연결된 '데이터 공급망(data supply chain)'을 신뢰 있게 구성한다.
- 셀프서브 플랫폼: 생성자가 자율적으로 양질의 데이터 제품을 만들도록 셀프서브 툴링을 제공한다.
- 연합형 거버넌스: 컨트랙트의 메타데이터로 보존·익명화·GDPR(General Data Protection Regulation) 대응 등을 자동화한다.
- 데이터 컨트랙트 없이는 사실상 데이터 메시를 구현할 수 없으며, 메시를 안 해도 컨트랙트는 유익하다.

```
[Domain A product]---contract--->[Domain B product]---contract--->[Business Value]
       (data as a product, self-serve, federated governance)
```

---

## Summary (핵심 정리)

- 데이터 컨트랙트는 생성자-소비자 간 합의된 인터페이스로, 기대치·거버넌스를 정의하고 품질 데이터의 명시적 생성을 가능케 하는 네 원칙으로 구성된다.
- 책임을 좌측(생성자)으로 이동시켜, 데이터를 가장 잘 아는 주체가 품질·신뢰성 문제를 원천에서 해결하게 한다.
- 데이터 메시와 상호보완적이며, 메시 도입 여부와 무관하게 신뢰할 수 있는 데이터 플랫폼 구축에 기여한다.
