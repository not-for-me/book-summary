# 17. Orchestration-Driven Service-Oriented Architecture

## 챕터 개요 (3줄 요약)

- 오케스트레이션 기반 SOA(Service-Oriented Architecture)는 1990년대 후반 전사적 재사용을 목표로 등장한 분산 스타일이다.
- 엄격한 서비스 분류(taxonomy)와 오케스트레이션 엔진으로 재사용을 추구했으나 강결합으로 사실상 실패했다.
- 기술 분할의 극단적 사례이자, 재사용=결합의 트레이드오프를 보여주는 역사적·교훈적 아키텍처다.

---

## 1. Topology / Style Specifics (토폴로지·세부)

> 기술적 책임이 다른 서비스 분류 체계를 중심으로 구성된 분산 아키텍처다.

자원(OS·DB 라이선스)이 비싸던 시대에 "가능한 한 재사용"이 지배 철학이었다. 잦은 M&A로 핵심 엔티티 중복·불일치 문제도 SOA를 매력적으로 만들었다.

### Taxonomy (분류 체계)

- Business services(비즈니스 서비스): 최상위 진입점, 굵은 입도(ExecuteTrade, PlaceOrder). "우리가 ~하는 사업인가?" 테스트 통과. 코드 없이 입출력·스키마만, 비즈니스 분석가가 정의.
- Enterprise services(엔터프라이즈 서비스): 미세 입도의 공유 구현(CreateCustomer, CalculateQuote). 재사용 자산을 점진적으로 축적하려는 빌딩 블록.
- Application services(애플리케이션 서비스): 일회성·단일 구현(예: geolocation).
- Infrastructure services(인프라 서비스): 모니터링·로깅·인증 등 운영 관심사.
- Orchestration engine & message bus: 아키텍처의 심장. 비즈니스/엔터프라이즈 서비스 매핑·트랜잭션 경계 정의, 통합 허브(ESB). Conway 법칙대로 통합 아키텍트 팀이 정치적 병목이 됨.

```
   Business Services (coarse, ExecuteTrade)
        | orchestration engine / message bus (ESB)
   Enterprise Services (fine, CreateCustomer)
   Application Services / Infrastructure Services
```

모든 요청은 오케스트레이션 엔진을 거친다(내부 호출도). 트랜잭션 입도 찾기가 어려워 실무에서 대부분 실패했다.

---

## 2. Reuse and Coupling (재사용과 결합)

> 재사용은 결합으로 구현되므로, 정규 서비스로의 통합은 막대한 결합을 낳는다.

보험사 6개 부서의 Customer를 단일 canonical Customer 서비스로 통합하면, Customer 변경이 모든 서비스에 파급되어 점진적 변경이 위험해진다(조율 배포·전체 테스트 필요). 또 자동차 보험의 운전면허처럼 한 부서만 필요한 세부사항을 모든 부서가 떠안게 된다(DDD가 전사적 재사용을 피하는 이유). "CatalogCheckout에 주소 줄 추가" 같은 작업이 여러 계층의 수십 서비스 변경을 요구한다.

---

## 3. Data / Cloud / Risks / Governance / Teams

> 데이터·팀 토폴로지는 당시 고려 대상이 아니었으며, 단일(혹은 소수) 관계형 DB를 썼다.

데이터: 트랜잭션을 메시지 버스의 선언적(declarative, XML) 처리에 위임(EntityBeans) — 런타임 트랜잭션 불확실성과 추상화 누수로 실패. 클라우드: SOA는 클라우드를 수십 년 앞섰으나, 현재는 클라우드·온프레미스 통합 아키텍처로 유용. 위험: 고비용·장기·유지보수 곤란, ESB가 전체를 잠식하는 Accidental SOA 안티패턴. 거버넌스: 당시 자동화 테스트·거버넌스 부재(수동 회의·리뷰). 현재는 통합 지점 간 데이터 누수를 막는 적합성 함수 유용. 팀: 극단적 책임 분리가 의사소통 안티패턴이 되어 team topologies 원칙을 낳음.

---

## 4. Style Characteristics (특성 평가)

> 역사상 가장 기술 분할된 범용 아키텍처로, 모놀리식과 분산의 단점을 모두 가진다.

분산이지만 단일 quantum(단일 DB + 오케스트레이션 엔진이라는 거대 결합점). 배포성·테스트성이 처참히 낮음(당시 목표가 아니었음). 탄력성·확장성은 벤더 노력으로 일부 지원(세션 복제). 성능은 요청이 여러 계층에 쪼개져 낮음. 단순성·비용 관계가 역전(복잡한데 비쌈). 기술 분할의 실무적 한계와 분산 트랜잭션의 어려움을 가르친 중요한 이정표.

```
  Deployability/Testability  * (disastrous)
  Single quantum (DB + orchestration = giant coupling point)
  Cost high, Simplicity low
```

---

## 5. Examples and Use Cases (예시)

> 1990년대 말~2000년대 초 대기업에서 성행했으나 microservices 등에 밀려났다.

엔티티 변경이 운 좋으면 엔터프라이즈 서비스만, 나쁘면 4~5계층 강결합 변경을 요구해 'change'를 두려워하게 만들었다. 현재는 ESB의 빌딩 블록(통합 허브+오케스트레이션 엔진)이 통합 아키텍처에 여전히 쓰임 — 엔터프라이즈 서비스를 통합 지점·패키지·맞춤 코드로 유연하게 구현 가능. 과거 접근에서 배워 유효한 부분은 쓰고 실패의 교훈은 내재화해야 한다.

---

## Summary (핵심 정리)

- 오케스트레이션 기반 SOA는 전사적 재사용을 위해 엄격한 서비스 분류와 오케스트레이션 엔진을 도입했다.
- 재사용=결합이라 강결합·기술 분할 극단화로 변경이 어려워 사실상 실패했고 microservices를 촉발했다.
- 분산이지만 단일 quantum의 단점을 모두 안았으며, ESB는 오늘날 통합 아키텍처에서 전략적으로 쓰인다.
