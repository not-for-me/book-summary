# 10. Consistency and Consensus

> Keywords: eventual vs strong consistency, linearizability(recency guarantee), linearizability vs serializability, strict serializability, CAP/PACELC theorem, CP/AP, ID generator, logical clock/Lamport clock/hybrid logical clock/vector clock, linearizable ID generator/timestamp oracle, consensus, single-value consensus/CAS/shared log(total order broadcast)/fetch-and-add/atomic commit, FLP, epoch/term number, Raft/Paxos/Zab/Viewstamped Replication, coordination service(ZooKeeper/etcd)

## 챕터 개요 (3줄 요약)
- fault tolerance를 위해 replication을 쓰면 inconsistency 위험이 생기며, eventual consistency(앱이 처리)와 strong consistency(시스템이 단일 노드처럼 동작) 두 철학이 있다.
- `linearizability`는 strong consistency의 정밀한 정의(recency guarantee — 단일 copy처럼 모든 연산이 atomic)이며, 비싸고 network 지연에 민감하다.
- fault-tolerant·linearizable replication은 `consensus`로 달성하며, CAS·shared log·atomic commit 등 여러 문제가 모두 consensus와 등가다.

---

## 1. Linearizability
> 단일 copy처럼 보이게 하는 recency guarantee — write 완료 후 모든 read는 최신값을 본다.

- 예: 스포츠 점수 — Bryce가 Aaliyah 뒤에 새로고침했는데 오래된 replica로 옛 점수를 보면 linearizability 위반.
- register(key/row/document)의 read/write/CAS를 시간선에 배치 — 한 read가 새 값을 보면 이후 모든 read도 새 값(화살표는 앞으로만).
- `linearizability ≠ serializability`: serializability는 transaction(다중 객체) isolation(순서 무관, stale read 허용), linearizability는 단일 객체 recency. 둘 다면 `strict serializability`(strong-1SR; Spanner, FoundationDB). CockroachDB는 serializable+일부 recency지만 strict 아님.

### 활용
- locking/leader election(lease는 linearizable이어야 split brain 방지 — ZooKeeper/etcd), uniqueness constraint(username/email — CAS 유사), cross-channel timing dependency(message queue+file storage race condition).

### 구현
- single-leader(잠재적 linearizable — 단 진짜 leader 확신 필요, async failover는 손실), consensus(likely — Zab/Raft, 단 leader 확인 없는 read는 stale), multi-leader(불가 — concurrent write 충돌), leaderless(probably not — quorum이어도 race condition, time-of-day LWW는 비linearizable).

---

## 2. The Cost of Linearizability (CAP)
> network partition 시 linearizability와 availability 중 선택해야 한다.

- `network partition`: region 간 단절. multi-leader는 각 region 독립 운영(나중 동기화). single-leader는 follower region이 leader 못 닿으면 write·linearizable read 불가(unavailable).
- `CAP theorem`: linearizability 필요+partition 시 unavailable(`CP`) vs 미필요+independent 처리(`AP`). 단 'consistency/availability/partition 중 2개'는 오해 — partition은 선택 아닌 fault. 'either consistent or available when partitioned'가 정확.
- CAP은 범위가 좁음(linearizability·partition만, <8% 사고). `PACELC`로 일반화(partition 없을 때도 latency vs consistency). 실무 가치 적음.
- linearizability를 안 쓰는 주 이유는 fault tolerance가 아니라 성능 — 멀티코어 RAM도 비linearizable(cache). Attiya-Welch: linearizable read/write 응답시간은 network 지연 불확실성에 비례.

---

## 3. ID Generators and Logical Clocks
> 단일 노드 autoincrement는 linearizable하나 fault-tolerant 아님 — 대안은 ordering이 약하다.

### ID Generator 옵션
- 단일 노드 autoincrement: linearizable(fetch-and-add)이나 SPOF·느림·bottleneck.
- sharded(짝/홀수), preallocated block, random UUID(v4, 128bit, 순서 없음), wall-clock timestamp(UUID v7/Snowflake/ULID — 근사 순서, 비linearizable).

### Logical Clocks
- `logical clock`: 시간 아닌 event를 세는 알고리즘. compact·unique·totally ordered·causality 일관.
- `Lamport clock`: (counter, node ID). 다른 노드 timestamp가 크면 로컬 counter를 그에 맞춤. total order지만 linearizability 아님(causal only).
- `hybrid logical clock`(HLC): 물리 시간 + Lamport ordering. 다른 노드 timestamp가 크면 전진, 항상 monotonic. 특수 hardware 불필요(CockroachDB). snapshot isolation의 transaction ID에 적합.
- `vector clock`: 노드별 counter — concurrent 여부 판별 가능(Lamport/HLC는 불가). 단 공간이 큼(노드당 integer).

### Linearizable ID Generator
- 비linearizable ID는 문제 유발(계정 private 변경 후 사진 업로드가 더 작은 ID → 무단 노출).
- 구현: 단일 노드가 atomic increment+persist+single-leader replication(`timestamp oracle`, Percolator). batch로 disk write 최적화(crash 시 skip은 OK). sharding/multi-region 어려움(단일 region).
- 또는 Spanner처럼 TrueTime 불확실성 구간 대기(통신 없이 linearizable, 단 hardware clock 필요).
- logical clock/ID generator로는 lock/uniqueness를 fault-tolerant하게 못 함(자기 timestamp가 최소인지 모든 노드에 물어야 — 장애 시 멈춤) → consensus 필요.

---

## 4. Consensus
> 여러 노드가 하나의 값(또는 순서)에 합의 — fault tolerance가 어려움의 원천.

- 알고리즘: Viewstamped Replication, Paxos, Raft, Zab(non-Byzantine). BFT 변형은 <1/3 Byzantine 허용(블록체인).
- `FLP`: asynchronous(clock 없음)·deterministic에선 consensus 항상 종료 불가. 단 timeout/random 허용 시 해결 가능 → 실무 가능.

### Many Faces of Consensus (모두 등가)
- `single-value consensus`(CAS 유사): uniform agreement·integrity·validity·termination. 3개(safety)는 단일 dictator로 쉬움, termination(liveness)이 fault tolerance — 과반(majority quorum) 정상 필요.
- `CAS`: null 초기화 후 CAS로 제안 → 등가.
- `shared log`(=total order broadcast/atomic broadcast): 모든 노드가 같은 순서로 entry를 읽음(eventual append·reliable delivery·append-only·agreement·validity). 첫 entry로 합의. state machine replication·event sourcing 기반.
- `fetch-and-add`: consensus number 2(2노드만). CAS·shared log는 ∞.
- `atomic commit`: consensus와 유사하나 abort 투표 있으면 반드시 abort(validity·nontriviality 추가). 등가.

### Consensus in Practice
- 대부분 shared log 제공(Raft/VR/Zab 기본, Paxos는 Multi-Paxos). DB replication(state machine replication), serializable transaction, fencing token(zxid) 등에 활용.
- single-leader → consensus: `epoch number`(Paxos ballot/Raft term/VR view)로 epoch당 leader 유일 보장. 두 라운드 투표(leader 선출 + 제안 append), quorum 겹침 필수. 2PC와 다름(아무 노드가 선거 시작, quorum만 필요).
- subtlety: 새 leader는 confirmed log entry를 honor해야(Raft는 up-to-date 노드만 leader). unclean leader election(Kafka)은 가용성↑이나 손실 위험. linearizable read도 quorum 확인 필요(etcd).
- 장단점: 'single-leader replication done right'(자동 failover·무손실·split brain 방지). 단 strict majority 필요(노드 추가는 느려짐), partition 시 minority 차단, timeout 튜닝 어려움, Raft edge case(pre-vote로 완화), EPaxos(leaderless).

### Coordination Services
- ZooKeeper/etcd/Consul(Chubby 모델): consensus + lock/lease·fencing(zxid/revision)·failure detection(ephemeral node/heartbeat)·change notification.
- 용도: leader election, shard→node 할당(rebalancing), configuration 관리, service discovery(단 consensus는 과함 — 캐싱·observer로 가용성 우선). 고정 소수 노드(3~5)로 'consensus outsourcing'. 빠른 변경 데이터엔 부적합(BookKeeper).

---

## Summary (핵심 정리)
- linearizability는 strong consistency의 정밀한 정의(단일 copy·atomic·recency)로 race condition 해소에 유용하나 느리다. 많은 replication이 겉보기와 달리 비linearizable.
- 단일 노드 autoincrement는 linearizable하나 fault-tolerant 아님. Lamport/HLC는 causal ordering 제공하나 linearizability는 아님.
- consensus가 fault-tolerant·linearizable replication을 가능케 함. CAS·lock/lease·uniqueness·shared log·atomic commit·fetch-and-add가 모두 consensus와 등가.
- Raft/Paxos는 'single-leader replication + 자동 leader election/failover'. 모든 write·linearizable read가 quorum(과반) 확인 필요 — 비싸지만 strong consistency+fault tolerance에 불가피.
- coordination service(ZooKeeper/etcd)는 consensus 위에 lock·fencing·failure detection·notification 제공. 단 strong consistency가 불필요하면 leaderless/multi-leader+logical clock이 낫다.
- 다음 연결: Ch11에서 batch processing을 다룸.