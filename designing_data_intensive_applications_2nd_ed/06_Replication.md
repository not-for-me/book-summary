# 06. Replication

> Keywords: single-leader/multi-leader/leaderless replication, sync vs async, semisynchronous, failover/split brain/fencing, statement-based/WAL/logical(row-based) log, change data capture, replication lag, eventual consistency, read-after-write/monotonic reads/consistent prefix reads, conflict resolution(LWW/CRDT/OT), sync engine/local-first, quorum(w+r>n)/read repair/hinted handoff/anti-entropy, happens-before, version vector

## 챕터 개요 (3줄 요약)
- replication = 같은 데이터를 여러 node에 복제 — 목적은 high availability, durability, disconnected operation, latency, read scalability.
- 변하는 데이터 처리가 핵심 난제이며, 알고리즘 3계열: single-leader / multi-leader / leaderless.
- async replication은 replication lag·conflict를 낳고, 이를 다루는 consistency 모델(read-after-write 등)과 conflict 해소(CRDT/version vector)가 핵심.

---

## 1. Single-Leader Replication
> 모든 write를 한 leader로 보내고, follower들이 같은 순서로 replication log를 적용한다.

### 기본 동작
- `leader`(primary/source)가 write 받아 로컬 저장 후 `replication log`(change stream)를 `follower`(read replica)에 전송. read는 아무 replica, write는 leader만. (master-slave 용어는 지양)
- PostgreSQL, MySQL, MongoDB, Kafka, Raft 기반(CockroachDB, etcd) 등 널리 사용. (backup은 replication과 별개 — 과거 시점 복원용으로 여전히 필요)

### Sync vs Async
- synchronous: follower 확인 후 성공 보고 — 최신 보장하나 그 follower 장애 시 write 차단. 그래서 보통 1개만 sync(`semisynchronous`).
- asynchronous: 응답 안 기다림 — leader 장애 시 미복제 write 손실(durability↓)하나 throughput·지리 분산에 유리.
- majority(quorum) 동기화도 가능(Ch10).

### Setup & 장애 처리
- 새 follower: lock 없이 leader의 consistent snapshot(특정 log 위치 연동: LSN/binlog/GTID) 복사 → snapshot 이후 변경분 따라잡기(catch up).
- follower 장애: 로그로 마지막 처리 지점 알고 catch-up recovery.
- leader 장애 `failover`: 장애 감지(timeout) → 새 leader 선출(consensus, 최신 follower) → 재구성. 위험: async 시 미복제 write 손실(GitHub MySQL/Redis 사고), `split brain`(두 leader), timeout 설정 곤란. `fencing`으로 구 leader 차단.

### Replication log 구현
- `statement-based`: SQL문 전달. nondeterministic(NOW/RAND), autoincrement, side effect 문제.
- `WAL shipping`: WAL을 그대로 전송. storage engine과 강결합 → 버전 불일치 불가(무중단 업그레이드 제약).
- `logical (row-based) log`: storage와 분리된 row 단위 log(MySQL binlog). 버전 호환·외부 파싱 용이 → `change data capture`(Ch12).

---

## 2. Problems with Replication Lag
> async follower는 stale할 수 있음 — eventual consistency가 낳는 anomaly를 consistency 모델로 보완.

- read-scaling: read를 다수 follower에 분산(async 필수). lag는 보통 1초 미만이나 부하·네트워크 시 수분.
- `eventual consistency`: 쓰기 멈추면 결국 수렴. '결국'은 의도적으로 모호.
- 3대 anomaly와 해법:
  - `read-after-write consistency`(read-your-writes): 자기가 쓴 건 반드시 봄. 해법 — 자기 데이터는 leader에서 읽기, 최근 쓴 후 1분은 leader, timestamp 추적. cross-device 시 metadata 중앙화·같은 region 라우팅.
  - `monotonic reads`: 시간 역행 금지(최신 본 뒤 더 옛것 안 봄). 해법 — 사용자별 같은 replica(user ID hash).
  - `consistent prefix reads`: causal 순서 보존(질문→답변). sharded DB에서 문제 — causal write를 같은 shard로, 또는 causal dependency 추적.
- 가장 단순한 해법은 강한 consistency(linearizability)+ACID DB(NewSQL) 사용.

---

## 3. Multi-Leader Replication
> 여러 node가 write를 받음(active/active). 주로 async, geo-distributed에 적합하나 conflict가 대가.

### Geo-distributed & 비교
- region마다 leader. 장점 — 로컬 write로 latency 숨김, region 장애·네트워크 문제 내성. 단점 — consistency가 약함(unique·잔액 음수 같은 제약 불가, 분산 시스템의 근본 한계).
- 토폴로지: all-to-all(SPOF 없음, 단 메시지 순서 역전 가능 — causality 문제), circular/star(한 node 장애가 전파 차단). 무한 loop 방지로 node id 태깅. 순서엔 version vector 필요.

### Sync Engine & Local-First
- 오프라인 동작 앱(calendar, Google Docs, Figma): 각 device/탭이 leader인 극단적 multi-leader. 로컬 즉시 반영 + async sync.
- `sync engine`: 변경 캡처·전송·merge·UI 갱신. `offline-first`(오프라인 편집), `local-first`(개발사 서비스 종료에도 동작, 개방 프로토콜 — 예: Git). 장점 — 빠른 UI, 오프라인, 단순 프로그래밍 모델. 한계 — 전체 데이터 사전 다운로드 필요(대용량 부적합).

### Conflict 해소
- `conflict avoidance`: 특정 record는 항상 같은 leader로(home region). 단 leader 변경 시 깨짐.
- `LWW`(last write wins): 최신 timestamp 채택 — concurrent write는 무작위 승자, 데이터 손실. clock 의존(logical clock으로 완화).
- manual resolution: sibling(B,C) 모두 저장 후 read 시 반환·병합. Amazon 장바구니 삭제 항목 재출현 같은 anomaly 주의.
- automatic resolution(`strong eventual consistency`): 모든 replica가 같은 상태로 수렴. text(삽입/삭제 보존), 집합, counter, key-value 별 merge.
- `CRDT`(conflict-free replicated datatype)와 `OT`(operational transformation): 자동 merge 두 계열. OT는 index 변환(Google Docs), CRDT는 문자별 불변 ID(Automerge, Yjs).
- subtle conflict: 회의실 중복 예약 등 — 단순 필드 충돌 아닌 제약 위반.

---

## 4. Leaderless Replication
> leader 없이 아무 replica나 write 수용. client가 여러 node에 병렬 read/write(Dynamo-style).

### 기본 & 복구
- Dynamo 영감(Riak, Cassandra, ScyllaDB). client/coordinator가 여러 replica에 write, 일부 ack로 성공. read도 병렬 — timestamp 큰 값 채택.
- 누락 복구: `read repair`(read 시 stale 발견하면 최신값 write-back), `hinted handoff`(다운 replica 대신 hint 저장 후 전달), `anti-entropy`(background로 차이 복사).

### Quorum
- n replica, write는 w개, read는 r개 ack. `w + r > n`이면 read가 최신값 포함(겹침 보장). 보통 n 홀수, w=r=(n+1)/2.
- w<n이면 일부 다운에도 write 가능, r<n이면 read 가능. n=5,w=3,r=3은 2개 다운 허용.
- 한계: 동시 read/write, 실패 write 미롤백, 노드 복구, rebalancing, clock 기반 LWW drop 등으로 최신 보장이 깨질 수 있음 — 확률 조정일 뿐. eventual consistency 'eventual' 정량화·모니터링 어려움.

```
n=3, w=2, r=2:  w+r=4 > 3  -> read set ∩ write set ≠ ∅ (최신값 1개 이상 보장)
```

### Performance & Multi-region
- 장점: failover 없음, 느린 replica 무시(`request hedging`으로 tail latency↓), gray failure에 강함.
- 단점: hinted handoff 부하, quorum 클수록 느린 replica 확률↑, 대규모 단절 시 quorum 불가(`sloppy quorum`로 우회).
- multi-region: coordinator가 로컬+타 region 1개에 전달. consistency level로 quorum 범위 선택(local quorum은 빠르나 stale 가능).

### Detecting Concurrent Writes
- 노드마다 도착 순서가 달라 영구 불일치 위험 → 수렴 위해 conflict 해소(LWW/manual/CRDT).
- `happens-before`: B가 A를 알/의존/기반하면 A→B(causal). 둘 다 서로 모르면 `concurrent`(물리 시간 무관, 상대성과 유비).
- 단일 replica: key별 version number 증가, read 시 sibling+version 반환, write 시 prior version 포함·merge. 서버는 version으로 overwrite/concurrent 판정(값 해석 불필요). 장바구니 예시로 causal dependency 포착.
- 다중 replica: replica별+key별 version → `version vector`(dotted version vector, Riak). overwrite vs concurrent 구분, 한 replica read 후 다른 replica write 안전. (version vector ≠ vector clock)

---

## Summary (핵심 정리)
- replication 목적: HA, durability, disconnected operation, latency, read scalability. 단순해 보이나 concurrency·fault 처리가 까다롭다.
- 3계열: single-leader(이해 쉽고 강한 consistency) / multi-leader·leaderless(faulty node·네트워크 내성↑, 단 conflict 해소·약한 consistency).
- async replication은 fault 시 손실 위험(failover 시 미복제 write). lag anomaly는 read-after-write·monotonic reads·consistent prefix reads 모델로 대응.
- multi-leader·leaderless는 version vector로 concurrent write 탐지, CRDT/LWW/manual로 merge해 수렴.
- 다음 연결: Ch7에서 단일 머신보다 큰 데이터를 위한 sharding(partitioning)을 다룸.