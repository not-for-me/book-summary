# 15. Event-Driven Architecture Style

## 챕터 개요 (3줄 요약)

- 이벤트 기반 아키텍처(EDA, Event-Driven Architecture)는 비동기·디커플링된 이벤트 처리 컴포넌트로 구성된 분산 스타일이다.
- 요청 기반 모델과 달리 '이미 일어난 일(event)'에 반응하며, 코레오그래피와 미디에이터 두 토폴로지가 있다.
- 성능·확장성·내결함성이 강점이나 워크플로우 제어·오류 처리·테스트가 어렵다.

---

## 1. Topology (토폴로지)

> 비동기 fire-and-forget 통신으로, 초기 이벤트·이벤트 브로커·이벤트 프로세서·파생 이벤트로 구성된다.

초기 이벤트(initiating event)가 브로커를 거쳐 한 프로세서에 전달되고, 프로세서는 작업 후 파생 이벤트(derived event)를 브로드캐스트한다. 다른 프로세서들이 이에 반응하며 연쇄된다(릴레이 경주 비유). 브로커는 보통 페더레이션되며 pub-sub(topic/stream) 모델을 쓴다. 파생 이벤트가 무한 반복되는 'poison event'를 주의해야 한다.

```
  Initiating Event -> [Broker] -> Processor A
       Processor A -> derived event -> [Broker]
       -> Notification / Payment / Inventory (parallel)
```

---

## 2. Style Specifics (스타일 세부)

> 이벤트(일어난 일, 응답 불필요, 다대일 아닌 일대다 브로드캐스트)와 메시지(명령/질의, 일대일, 응답 필요)는 다르다.

### Events vs Messages

이벤트는 topic/stream으로 일대다 브로드캐스트, 메시지는 queue로 일대일 전달. 파생 이벤트는 EDA의 핵심이며 한 프로세서가 여러 파생 이벤트를 만들 수 있다. 아무도 안 듣는 'extensible event'도 미래 확장 hook으로 유용하다.

### Asynchronous Capabilities (비동기 능력)

응답성(responsiveness, 사용자에게 접수 알림)과 성능(performance, 실제 처리 속도)을 구분한다. 비동기로 댓글 게시 시 사용자 체감은 25ms(동기 3,100ms 대비). 비동기는 동기 결합으로 두 quantum이 엉키는 'Dynamic Quantum Entanglement'를 풀어 독립 quantum으로 만든다(주식 거래 예시).

### Event Payload (이벤트 페이로드)

- Data-based(데이터 기반): 모든 정보 포함. 성능·확장성 좋음, DB 조회 불필요. 단 데이터 일관성·계약 관리·버저닝·stamp coupling·대역폭 문제.
- Key-based(키 기반): 키만 포함, 프로세서가 DB 조회. 데이터 일관성·계약 단순·대역폭 좋음. 단 성능·확장성 저하, DB 부하.
- Anemic event(빈약한 이벤트): 정보 부족으로 처리 불가한 이벤트(피해야 함, 변경 전후 값 포함).

```
  Key-based <-------- spectrum --------> Data-based
  (only key)     (right granularity)    (all data)
```

### The Swarm of Gnats Antipattern (각다귀 떼 안티패턴)

한 프로세서가 너무 많은 미세 파생 이벤트를 트리거해 시스템을 포화시키는 안티패턴. 결과/상태 변화에 초점을 맞춰 적절한 입도로 통합한다(coarse-grained도 비효율, fine-grained 과다도 문제).

### Error Handling — Workflow Event Pattern

이벤트 컨슈머가 오류 시 즉시 Workflow Processor에 위임하고 다음 메시지로 이동(응답성 유지). 워크플로우 프로세서가 자동 수정 후 재제출하거나, 불가 시 사람 대시보드로 보냄(주식 거래 "SHARES" 오류 예시). 단 메시지 순서가 어긋날 수 있어 FIFO 큐로 관리.

### Preventing Data Loss (데이터 손실 방지)

AMQP(Advanced Message Queuing Protocol) 기반. Event Forwarding 패턴: (1) 영속 큐+동기 send로 producer-큐 손실 방지, (2) client acknowledge mode로 컨슈머 크래시 시 보존, (3) ACID 커밋+LPS(Last Participant Support)로 DB 영속 보장.

### Request-Reply Processing (요청-응답)

동기가 필요할 때 pseudosynchronous. request/reply 두 큐 사용. 구현: (1) Correlation ID(CID, 권장), (2) 임시 큐(단순하나 브로커 부하).

### Mediated EDA (미디에이터 토폴로지)

이벤트 미디에이터가 워크플로우를 제어(메시지=명령 사용). 도메인별 다중 미디에이터로 단일 장애점 회피. 구현: 단순(Apache Camel, Mule), 복잡 조건부(BPEL: Apache ODE, Oracle BPEL), 사람 개입 장기 트랜잭션(BPM 엔진 jBPM). 분류(simple/hard/complex) 후 위임 모델 권장. 트레이드오프: 워크플로우 제어·오류 처리 vs 성능·확장성.

---

## 3. Data Topologies (데이터 토폴로지)

> 모놀리식·도메인·전용(dedicated) 세 가지가 있으며 트레이드오프가 크다.

- Monolithic: 모든 프로세서가 중앙 DB 직접 조회(디커플링 유지, 동기 통신 불필요). 단 내결함성·확장성·변경 제어 약함, 단일 quantum.
- Domain: 도메인별 DB. 내결함성·확장성·변경 제어 개선. 단 도메인 간 동기 호출 필요할 수 있음.
- Dedicated(database-per-service): 프로세서별 DB. 최고 내결함성·확장성. 단 비용·프로세서 간 동기 결합. 자기완결적 프로세서에 적합.

---

## 4. Cloud / Risks / Governance / Teams

> 디커플링 특성으로 클라우드와 잘 맞으나, 여러 위험과 거버넌스가 필요하다.

위험: 비결정적 처리의 부작용, 과도한 정적 결합(계약), 과도한 동기 통신(EDA 부적합 신호), 상태 관리 어려움(초기 이벤트 완료 시점 파악 곤란). 거버넌스: 비구조적이며 관찰성(로그) 필요. 정적 결합(계약 변경률·stamp coupling)과 동적 결합(동기 호출 추적)을 모니터링. 팀: 기술 분할. complicated-subsystem·platform 팀에 적합, stream-aligned·enabling 팀은 디커플링 때문에 어려움.

---

## 5. Style Characteristics (특성 평가)

> 성능·확장성·내결함성이 강점(4~5점), 단순성·테스트성은 낮다.

quanta는 1~다수(공유 DB나 request-reply는 단일 quantum). 성능(비동기+병렬), 확장성(competing consumers/consumer groups로 프로그래밍적 로드밸런싱), 내결함성(eventual consistency) 모두 높음. 진화성 5점(파생 이벤트 hook). 단점: 워크플로우 제어 곤란, 오류 처리·복구 어려움, 비결정적 워크플로우라 테스트 곤란.

```
  Performance/Scalability/Fault Tolerance ****~*****
  Evolvability *****
  Simplicity/Testability  low
```

---

## 6. Choosing & Examples (선택과 예시)

> 요청 기반은 구조적·데이터 주도 요청에, 이벤트 기반은 유연·동작 기반·고응답성에 적합하다.

이벤트 기반 장점: 동적 콘텐츠 대응, 확장성·탄력성, 민첩성, 적응성, 실시간 의사결정. 트레이드오프: eventual consistency만 지원, 흐름 제어·결과 확실성 부족, 테스트·디버그 곤란. 예시: 주문 시스템(병렬 디커플링), Going Going Gone 경매(입찰=요청이 아닌 '일어난 이벤트', Bid Capture→Auctioneer/Bid Streamer/Bidder Tracker). 처리 대부분이 요청 기반이면 마이크로서비스를 고려한다.

---

## Summary (핵심 정리)

- EDA는 비동기·디커플링 이벤트 프로세서로 고성능·고확장·내결함 시스템을 만든다.
- 이벤트/메시지 구분, 페이로드(data/key-based), Swarm of Gnats·anemic event 같은 함정을 관리해야 한다.
- 코레오그래피(고성능)와 미디에이터(워크플로우 제어)를 택하며, 워크플로우 제어·오류 처리·테스트가 난제다.
