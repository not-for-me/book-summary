# 09. The Trouble with Distributed Systems

> Keywords: partial failure, unreliable network/asynchronous packet network, timeout/unbounded delay, network congestion/queueing, circuit vs packet switching, unreliable clocks, time-of-day vs monotonic clock, NTP/clock drift/confidence interval, TrueTime, process pause/GC pause, quorum/majority, distributed lock/lease/fencing token, Byzantine fault, system model(synchronous/partial/async), safety vs liveness, formal methods/model checking/deterministic simulation testing

## 챕터 개요 (3줄 요약)
- 분산 시스템의 정의적 특징은 `partial failure`(일부만 비결정적으로 고장) — 단일 컴퓨터의 이상적 결정성과 근본적으로 다르다.
- network(임의 지연/손실), clock(drift/jump), process pause(GC/VM suspend) 모두 신뢰할 수 없어 노드는 다른 노드 상태를 확신할 수 없다.
- 해법의 토대: 단일 노드를 믿지 말고 quorum(다수결)에 의존하고, fencing token으로 zombie를 막으며, system model로 가정을 형식화한다.

---

## 1. Faults and Partial Failures
> 단일 컴퓨터는 결정적(작동 or 완전 고장)이나, 분산은 일부가 예측 불가하게 고장난다.

- 컴퓨터는 내부 fault 시 잘못된 결과보다 crash를 택해 이상적 모델 제공. 분산은 물리적 현실(network)에 직면.
- `partial failure`: 일부는 정상, 일부는 비결정적 고장 — 성공 여부조차 모를 수 있음. 이를 견디면 rolling upgrade 등 가능(불신·비관·편집증이 보상).

---

## 2. Unreliable Networks
> shared-nothing은 network로만 통신. asynchronous packet network는 도착 시점·여부를 보장 안 함.

- 요청 무응답 시 구분 불가: 요청 손실 / 노드 다운 / 응답 손실 / 지연. 유일한 수단은 `timeout`(그래도 처리 여부 모름).
- `TCP`: 재전송·순서·checksum·congestion control 제공('reliable')하나, error로 닫히면 처리량 모름. application 응답이 있어야 성공 확신.
- network fault는 실무에서 흔함(월 12회, 사람 실수·misconfiguration이 주원인, 비대칭 fault·partial interruption). `network partition`(netsplit)은 sharding과 무관.
- fault detection: RST/FIN, ICMP unreachable 등 빠른 피드백은 유용하나 의존 불가. timeout으로 false positive/negative 균형.

### Timeout & Queueing
- timeout 길면 회복 느림, 짧으면 오판(일시 slowdown을 죽음으로) → cascading failure 위험. asynchronous network는 unbounded delay.
- 지연 변동의 주원인은 `queueing`: switch 혼잡, 목적지 CPU 대기, VM pause, TCP 송신 buffer. `Phi Accrual failure detector`로 동적 timeout 조정.

### Sync vs Async Network
- 전화망은 `circuit`(고정 대역 예약, bounded delay)이라 동기적. Ethernet/IP는 `packet switching`(bursty 트래픽 최적화, queueing→unbounded delay).
- 변동 지연은 자연법칙이 아니라 `dynamic resource partitioning`의 cost/benefit trade-off(정적 분할=예측 가능하나 비싸고 저활용).

---

## 3. Unreliable Clocks
> 각 머신의 quartz clock은 부정확. NTP 동기화도 한계가 많다.

### Time-of-day vs Monotonic
- `time-of-day clock`(wall-clock, epoch 이후): NTP 동기화하나 뒤로 점프·leap second·DST로 elapsed time 측정 부적합.
- `monotonic clock`(항상 전진): duration/timeout 측정용. 절대값 무의미(머신 간 비교 불가). NTP는 slewing으로 속도만 조정.

### 동기화 정확도 & 위험
- quartz drift(~200ppm), NTP 강제 reset(시간 점프), 방화벽 차단, network delay(최소 35ms 오차), 잘못된 NTP 서버, leap second(시스템 crash 사례), VM clock 점프.
- 고정밀(MiFID II 100μs)은 GPS/atomic clock+PTP로 가능하나 care 필요. 잘못된 clock은 조용히 데이터 손실 → offset 모니터링·dead 선언 필수.

### Clock 의존의 위험
- event ordering: time-of-day로 순서 매기면 causal하게 늦은 write가 더 작은 timestamp 가능 → `LWW`로 write 소실(Cassandra/ScyllaDB). version vector·logical clock이 안전.
- `confidence interval`: clock 읽기는 점이 아니라 [earliest, latest] 구간. 대부분 노출 안 함. 예외 — Google Spanner `TrueTime`, Amazon ClockBound.
- global snapshot: Spanner는 TrueTime 구간이 겹치지 않게 commit 시 구간 길이만큼 대기 → causality 보장(GPS/atomic clock으로 ~7ms 불확실성).

---

## 4. Process Pauses
> thread가 임의 시점에 오래 멈출 수 있어, lease 기반 leader 코드가 위험하다.

- lease(timeout 있는 lock) 갱신 코드의 결함: 동기 clock 의존, check~process 사이 pause 가정.
- pause 원인: lock contention, `GC pause`(과거 수분), VM suspend/live migration, laptop sleep, context switch/steal time, 동기 disk I/O, page fault/swapping(thrashing), SIGSTOP.
- 분산 시스템엔 단일 머신의 mutex/semaphore가 안 통함(shared memory 없음). 노드는 임의 시점 pause를 가정해야.
- hard real-time(RTOS)은 보장 가능하나 매우 비싸고 throughput 낮음(safety-critical 임베디드용). GC pause는 튜닝·pool·off-heap·rolling restart로 완화.

---

## 5. Knowledge, Truth, and Lies
> 노드는 message로만 추론 — 자기 판단을 믿지 말고 quorum과 fencing에 의존한다.

### Majority / Quorum
- 비대칭 fault·pause로 멀쩡한 노드가 dead로 오판될 수 있음. 단일 노드 의존 금지 → `quorum`(다수결, 과반). 과반은 동시에 둘일 수 없어 안전.

### Distributed Lock & Fencing
- lease 만료를 모르는 `zombie`(former leaseholder)나 지연된 요청이 파일을 손상(HBase 실제 버그).
- `fencing token`: lock 부여 시 증가하는 번호. storage가 더 큰 token을 본 뒤엔 작은 token write 거부 → zombie 차단(Chubby sequencer, Kafka epoch, Paxos ballot, Raft term). conditional write(CAS)/S3 conditional write로도 가능. 다중 replica는 token을 timestamp 상위 bit에.
- STONITH(노드 강제 종료)는 network delay·동시 종료 위험으로 불완전.

### Byzantine Faults
- 노드가 '거짓말'(임의/악의 응답)하는 경우 = `Byzantine fault`(Byzantine Generals Problem). 항공우주·블록체인엔 관련하나 일반 server-side는 노드를 신뢰(같은 조직·낮은 방사선)해 BFT 불필요(비쌈).
- 약한 거짓말 방어: application 레벨 checksum, 입력 sanitization, 다중 NTP 서버.

### System Model & Correctness
- timing 모델: synchronous(bounded, 비현실적), `partially synchronous`(대부분 bounded, 때때로 초과 — 현실적), asynchronous(clock 없음, 제한적).
- node 모델: crash-stop, `crash-recovery`(stable storage 생존), limping/gray failure/fail-slow, Byzantine. 실용은 partially synchronous + crash-recovery.
- `safety`(나쁜 일 안 일어남, 위반은 되돌릴 수 없음 — 항상 보장) vs `liveness`('결국' 좋은 일 — caveat 허용). eventual consistency는 liveness.

### Formal Methods & Testing
- formal verification(수학적 증명), `model checking`(TLA+ 등으로 상태 공간 탐색 — 실제 코드 아닌 모델), `fault injection`(Chaos Monkey, Jepsen — 실행 중 fault 주입), `deterministic simulation testing(DST)`(실제 코드를 결정적 mock으로 재현·재현 가능, FoundationDB/TigerBeetle/Antithesis).
- determinism의 힘: event sourcing replay, durable execution, state machine replication 모두 결정성에 의존.

---

## Summary (핵심 정리)
- 분산 시스템의 핵심 문제: network packet 손실/지연, clock 비동기·점프, process pause로 인한 partial failure. 무응답 시 원인 구분 불가.
- fault 탐지조차 어렵고(timeout은 network/node 고장 구분 못 함, limping node는 더 어려움), 공유 상태가 없어 단일 노드가 안전하게 결정 못 함 → quorum 필요.
- 단일 머신으로 해결 가능하면 그게 낫지만(embedded engine), fault tolerance·low latency는 분산이 필수. network/clock 불확실성은 비용을 들이면 줄일 수 있으나 대부분 싸고 불안정한 쪽을 택함.
- system model(partial synchrony + crash-recovery)과 safety/liveness로 알고리즘 정확성을 추론하고, formal method·DST로 검증.
- 다음 연결: Ch10에서 이 문제들을 극복하는 consistency·consensus 알고리즘을 다룸.