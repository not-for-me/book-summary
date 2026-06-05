# 10. Data Contracts in Practice

## 챕터 개요 (3줄 요약)

- 데이터 컨트랙트 설계는 목적 식별 → 트레이드오프 고려 → 정의 → 배포의 네 단계 반복 과정이다.
- 컨트랙트 정의, 데이터 품질, 성능·신뢰성 세 영역에서 모니터링·강제(enforcement)를 구현한다.
- 소스 시스템과 컨트랙트 간 일관성이 필요할 때 트랜잭셔널 보장을 위한 발행(publishing) 패턴을 선택한다.

---

## 1. Designing a Data Contract

> 데이터 컨트랙트 설계는 생성자와 소비자의 협업을 통한 네 단계 반복 과정이다.

- 목적 식별(identifying the purpose): 누구를 위한 데이터인지, 어떤 문제·비즈니스 가치를 다루는지 묻고 소비자 요구를 수집한다.
- 트레이드오프 고려(considering the trade-offs): 생성자가 비용·성능·서비스 영향 등 제약을 검토하므로 컨트랙트를 소유해야 한다.
- 논의 내용은 RFC(Request For Comment) 같은 문서에 기록해 결정을 검토 가능하게 한다.
- 컨트랙트 정의(defining): 스키마·필드·문서, 데이터 품질 체크, SLO(Service-Level Objective), 거버넌스 분류를 명시한다.
- SLO는 생성자가 감당 가능하고 소비자 최소 요구를 넘지 않는 수준으로 설정한다(과도하면 비용 낭비).
- 배포(deploying): Git 머지로 인터페이스·서비스가 프로비저닝되며, 과거 데이터 백필과 모니터링 설정이 필요할 수 있다.

---

## 2. Monitoring and Enforcing Data Contracts

> 기대치를 설정했으면 충족을 입증해야 하므로, 정의·데이터 품질·성능 세 영역에서 모니터링·강제를 구현한다.

### The data contract's definition

- 컨트랙트는 사람이 작성해 오류가 가능하므로 자동 검증으로 버전·소유자·문서·필드(이름·타입·설명) 존재를 확인한다.
- 개인 데이터 필드에 익명화 전략이 정의됐는지 등 거버넌스 체크를 추가한다.
- CUE 같은 도구로 YAML/JSON 컨트랙트를 검증하고, 스키마 레지스트리로 파괴 변경(스키마 진화)을 막는다.
- 이 검증들을 CI(Continuous Integration) 파이프라인에서 실행해 무효 컨트랙트의 운영 배포를 차단한다.

### The quality of the data

- 두 종류 품질 이슈: 사전에 예측·테스트한 것(스키마, 범위 검증)과 예측하지 못한 것(분포 이상 등).
- 예측 못한 이슈 방어: 장애에 탄력적인 애플리케이션, 데드 레터 큐(dead letter queue), 데이터 옵저버빌리티(observability) 도구.
- 품질 체크 구현 위치 세 곳: 발행 시점(publishing time), 인프라(infrastructure), 발행 후(after publishing).
- 발행 시점·인프라 체크는 소스에서 이슈를 차단해 소비자 영향을 줄인다.
- 인프라 체크는 웨어하우스 테이블·Kafka·Pub/Sub의 스키마로 가장 쉽게 구현되나 기본 타입만 검사한다.
- 발행 후 체크는 Great Expectations, Soda, dbt 등으로 하되 이미 소비됐을 수 있어 영향이 크다.

```
[Source] --publishing time check--> [Interface] --after publishing check--> [Consumers]
              (+ infrastructure schema check at the interface)
```

### Performance and dependability

- SLO(완전성·적시성·가용성)를 컨트랙트에 정의해 문서화하고 메트릭 수집·보고를 자동화한다.
- Pub/Sub 예시: publishTime에서 generationTime을 빼 적시성(timeliness)을 근실시간으로 계산·알림한다.
- 이 모니터링 서비스는 완전성·가용성 등 다른 SLO로도 확장 가능하다.

```
slos:
  completeness_percent: 100
  timeliness_mins: 60
  availability_percent: 95
```

---

## 3. Data Contract Publishing Patterns

> 핵심 고려사항은 소스 시스템과 인터페이스 간 트랜잭셔널 보장(transactional guarantee) 필요 여부다.

- 두 작업(서비스 DB 쓰기, 인터페이스 발행)을 원자적(atomic)으로 만들어야 일관성이 보장된다.
- 강한 일관성이 불필요하면 복잡성을 피하고 성능을 높일 수 있으니 매 사용 사례마다 필요성을 따진다.

### Writing directly to the interface

- 가장 단순하며 라이브러리로 쉽게 쓸 수 있으나, 웨어하우스의 단건 쓰기는 느릴 수 있다(배치는 양호).
- 트랜잭셔널 보장 패턴이 없으므로 일관성이 필요하면 부적합하다.

### Materialized views on CDC

- CDC(Change Data Capture)가 DB 변경을 캡처해 트랜잭셔널 보장을 제공하며, 생성자가 변환 뷰를 소유한다.
- 단, 생성자가 두 서비스(본 서비스 + 변환)와 익숙하지 않은 도구(dbt, Flink, Benthos)를 유지해야 해 비권장된다.

### The transactional outbox pattern

- 애플리케이션 DB에 outbox 테이블을 두고 변경과 이벤트를 동일 트랜잭션에서 써 일관성을 보장한다.
- 별도 프로세스가 outbox에서 이벤트를 읽어 인터페이스로 발행한다.
- 이벤트 구조는 DB 구조와 분리되어 생성자가 자율적으로 DB를 변경할 수 있다.
- 단점: DB에 추가 쓰기 부하와 폴링 프로세스가 발생한다.

### The listen-to-yourself pattern

- 1차 쓰기를 보조 데이터스토어(이벤트 스트리밍/메시지 브로커)에 하고, 나중에 DB와 인터페이스에 복제한다.
- DB 이중 쓰기가 없어 부하가 줄고, 테이블 간 트랜잭션 지원이 약한 DB에 유용하다.
- 단점: 애플리케이션 DB가 최종 일관성(eventually consistent)이라 즉시 조회 시 최신 변경이 누락될 수 있다.
- 메시지 순서 어긋남·중복 가능성이 있어 순서 보장 브로커나 멱등(idempotent) 쓰기가 필요할 수 있다.

```
Outbox:        [DB tx: app tables + outbox] -> poller -> [interface]
Listen-to-self:[write to broker] -> consumer1 -> app DB / consumer2 -> interface
```

---

## Summary (핵심 정리)

- 데이터 컨트랙트는 목적 식별·트레이드오프·정의·배포의 단계로 생성자-소비자 협업을 통해 설계한다.
- 정의·데이터 품질(예측/비예측)·성능을 아키텍처 여러 지점에서 모니터링·강제해 기대 충족을 입증한다.
- 일관성이 필요하면 트랜잭셔널 아웃박스나 listen-to-yourself 패턴으로 발행하며, 이로써 데이터 품질을 데이터 컨트랙트로 구동한다.
