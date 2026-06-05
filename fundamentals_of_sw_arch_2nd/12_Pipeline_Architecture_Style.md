# 12. Pipeline Architecture Style

## 챕터 개요 (3줄 요약)

- 파이프라인(pipes and filters) 아키텍처는 기능을 단일 목적 필터로 나누고 파이프로 연결하는 기술 분할 스타일이다.
- Unix 셸·함수형 언어·MapReduce의 기반 원리이며, 결정적·순차적 단방향 처리에 적합하다.
- 단순·저비용·모듈성이 강점이나 모놀리식이라 확장성·내결함성은 낮다(quantum=1).

---

## 1. Topology (토폴로지)

> 두 컴포넌트 유형, 즉 필터(filter)와 파이프(pipe)로 구성되며 단일 배포 단위다.

필터는 비즈니스 기능을 수행하고, 파이프는 데이터를 다음 필터로 전달한다. 파이프는 단방향·점대점(point-to-point) 통신이다.

```
  Producer --pipe--> Transformer --pipe--> Tester --pipe--> Consumer
```

---

## 2. Style Specifics (스타일 세부)

> 보통 모놀리식이나 각 필터를 서비스로 배포해 분산화할 수도 있다.

### Filters (필터)

자기완결적·무상태(stateless)이며 단일 작업만 수행한다. 다중 클래스로 구현될 수 있어 컴포넌트로 간주된다. 네 가지 유형: Producer(시작점, source), Transformer(데이터 변환=map), Tester(조건 검사=reduce, 진행 여부 결정), Consumer(종료점, DB 저장/UI 표시). 단방향·단순성이 합성 재사용(compositional reuse)을 촉진한다(Unix 셸 예: Knuth의 10페이지 Pascal vs McIlroy의 6줄 셸 스크립트).

### Pipes (파이프)

필터 간 통신 채널. 단방향·점대점. 작은 데이터를 선호(고성능). 분산 시 REST·메시징·스트리밍 등 원격 호출. 모놀리식/분산 모두 동기·비동기 가능(모놀리식은 스레드·내장 메시징).

---

## 3. Data / Cloud / Risks / Governance / Teams

> 보통 단일 모놀리식 DB이나, 필터별 DB도 가능하다.

데이터: 단일 DB ~ 필터당 DB까지 다양(연속 적합성 함수 예시). 클라우드: 모듈성 덕에 적합. AWS Step Functions(각 필터=람다, Standard/Express 워크플로우), 서버리스/컨테이너 배포 가능. 위험: 필터에 책임 과부하, 양방향 통신 도입(스타일 부적합 신호), 오류 처리·종료 어려움, 필터 간 계약(contract) 변경 관리. 거버넌스: 태그(Java 어노테이션, C# 커스텀 속성)로 필터 유형(PRODUCER/TESTER/TRANSFORMER/CONSUMER)을 표시해 책임 과부하 방지. 팀: 어떤 구성과도 잘 맞음(enabling 팀이 흐름 방해 없이 필터 추가 가능).

---

## 4. Style Characteristics (특성 평가)

> 비용·단순성·모듈성이 강점이나 모놀리식이라 탄력성·확장성은 1점이다.

기술 분할이며 quantum=1. 모듈성은 필터 분리로 달성(한 필터를 다른 필터 영향 없이 교체). 배포성·테스트성은 계층형보다 약간 높음. 탄력성·확장성·내결함성은 낮으나, 비동기 분산화로 개선 가능(단 비용·단순성 희생 — 전형적 트레이드오프).

```
  Cost/Simplicity/Modularity ***** 
  Deployability/Testability  ***
  Scalability/Elasticity/FT  *
```

---

## 5. When to Use / Not (사용 시점)

> 구별되고 순서가 있으며 결정적인 단방향 처리 단계가 있는 시스템에 적합하다.

빠듯한 시간·예산에도 적합. 부적합: 높은 확장성·탄력성·내결함성 필요(분산화로 완화), 필터 간 양방향 통신, 비결정적 워크플로우(이 경우 event-driven이 나음). 예시: EDI 도구, ETL(Extract Transform Load) 도구, Apache Camel 오케스트레이터/중재자. 사례: Kafka 텔레메트리 처리(Service Info Capture→Duration tester→Duration Calculator / Uptime tester→Uptime Calculator→MongoDB). 확장이 쉬워 새 tester 필터를 쉽게 추가할 수 있다.

---

## Summary (핵심 정리)

- 파이프라인은 무상태 단일 목적 필터(Producer/Transformer/Tester/Consumer)를 단방향 파이프로 연결한다.
- 단순·저비용·모듈성이 강점이며 결정적·순차 처리(ETL·EDI)에 적합하다.
- 모놀리식이라 확장성·내결함성은 약하나 비동기 분산화로 개선 가능(비용 트레이드오프).
