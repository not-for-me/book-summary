# 08. Transactions

> Keywords: ACID(atomicity/consistency/isolation/durability), BASE, dirty read/write, read skew, lost update, write skew/phantom, isolation level(read committed/snapshot isolation=repeatable read/serializable), MVCC, conditional write(CAS)/optimistic locking, serializability(serial execution/2PL/SSI), predicate lock/index-range lock, distributed transaction, atomic commit/2PC, XA, exactly-once/idempotence

## 챕터 개요 (3줄 요약)
- transaction은 여러 read/write를 하나의 logical unit(commit 또는 abort)으로 묶어 partial failure·concurrency를 application으로부터 숨기는 추상화다.
- 약한 isolation level은 성능은 좋으나 dirty read/lost update/write skew 등 race condition을 남기며, serializability만이 모든 race condition을 막는다.
- serializability는 serial execution/2PL/SSI 세 방식으로 구현되고, 여러 node에 걸친 atomicity는 2PC(atomic commit)로 달성하나 XA는 운영 문제가 많다.

---

## 1. What Is a Transaction? / The Meaning of ACID
> transaction은 자연법칙이 아니라 프로그래밍 모델 단순화를 위한 safety guarantee다.

- 1975년 IBM System R 이래 거의 동일. NoSQL이 한때 transaction을 버렸으나 NewSQL(CockroachDB, TiDB, Spanner, FoundationDB)이 sharding+consensus로 확장 가능함을 증명.
- `ACID`: Atomicity(부분 실패 시 abort·rollback, 'abortability'), Consistency(application 불변식 invariant — DB만의 속성 아님, app 책임), Isolation(동시 transaction 격리, 이상은 serializability), Durability(commit 후 손실 없음 — fsync/WAL/checksum/replication, 단 완벽한 durability는 없음).
- `BASE`(basically available, soft state, eventual consistency)는 사실상 'not ACID'.

### Single vs Multi-object
- multi-object transaction: 여러 record를 sync(denormalized counter, FK, secondary index). BEGIN~COMMIT으로 그룹화.
- single-object: storage engine이 log(atomicity)+lock(isolation)으로 보장. atomic increment, conditional write(CAS).
- error 시 abort+retry가 기본 철학(leaderless는 best-effort). retry 주의: 성공했는데 ack 손실 시 중복(idempotence 필요), 과부하 시 악화, 영구 error는 무의미, 외부 side effect.

---

## 2. Weak Isolation Levels
> serializability는 비싸 약한 level이 흔히 쓰이나, 미묘한 race condition을 남긴다(금전 사고 사례 다수).

### Read Committed
- 보장: no dirty read(uncommitted 데이터 안 봄), no dirty write(uncommitted 값 안 덮음). Oracle/PostgreSQL/SQL Server 기본.
- 구현: write는 row-level lock, read는 lock 대신 old committed 값 반환(read lock은 long write가 read를 막아 부적합).
- `read uncommitted`: dirty write만 방지(dirty read 허용).

### Snapshot Isolation (= Repeatable Read)
- `read skew`(non-repeatable read): transaction 중 DB의 inconsistent 상태를 봄($100 증발). backup·analytics에 치명.
- snapshot isolation: 각 transaction이 시작 시점의 consistent snapshot에서 read. 'readers never block writers, writers never block readers'.
- `MVCC`(multiversion concurrency control): row마다 여러 committed version(txid의 inserted_by/deleted_by). visibility rule로 일관 snapshot 제공. immutable B-tree(copy-on-write, CouchDB/Datomic/LMDB)도 한 방법.
- 명명 혼란: PostgreSQL은 snapshot isolation을 'repeatable read', Oracle은 'serializable'로 부름. SQL 표준의 isolation level 정의는 모호.

### Preventing Lost Updates
- `lost update`: read-modify-write를 동시 실행해 한 수정이 사라짐(counter, JSON 수정, wiki 동시 편집).
- 해법: atomic write op(`UPDATE ... SET value=value+1`), explicit lock(`SELECT FOR UPDATE`), 자동 lost update 탐지(PostgreSQL repeatable read 등, MySQL InnoDB는 미탐지), conditional write(CAS, `optimistic locking`).
- 복제 DB(multi-leader/leaderless)는 단일 최신 copy 가정이 깨져 lock/CAS 부적합 → sibling+CRDT(commutative 연산은 lost update 방지). LWW는 lost update에 취약.

### Write Skew and Phantoms
- `write skew`: 두 transaction이 같은 객체를 read하고 서로 다른 객체를 write해 invariant 위반(on-call 의사 둘 다 동시에 off-call → 0명). lost update의 일반화.
- 예: 회의실 중복 예약, 게임 같은 위치, username 중복, double-spending.
- `phantom`: 한 transaction의 write가 다른 transaction의 search query 결과를 바꿈. 행이 없으면 `SELECT FOR UPDATE`가 lock 못 검 → `materializing conflicts`(lock용 행 미리 생성)는 최후 수단. 진짜 해법은 serializable.

```
Isolation level   | dirty read | read skew | phantom | lost update | write skew
Read uncommitted  | possible   | possible  | possible| possible    | possible
Read committed    | prevented  | possible  | possible| possible    | possible
Snapshot isolation| prevented  | prevented | prevented| depends    | possible
Serializable      | prevented  | prevented | prevented| prevented  | prevented
```

---

## 3. Serializability
> 가장 강한 level — 동시 실행이 어떤 serial 순서와 동일한 결과를 보장(모든 race condition 방지). 3가지 구현.

### Actual Serial Execution
- 단일 thread로 한 번에 하나씩 실행(VoltDB, Redis, Datomic). RAM 저렴+OLTP가 짧다는 점에서 가능.
- 조건: transaction을 `stored procedure`로 미리 제출(대화형 X, 네트워크 대기 제거), in-memory, 결정적. throughput은 단일 core 한계 → sharding(단 cross-shard는 느림).

### Two-Phase Locking (2PL)
- 30년간 표준. shared/exclusive lock — writer가 reader도 차단(snapshot isolation과 반대). 모든 race condition 방지하나 성능·동시성 저하, deadlock 빈번.
- `predicate lock`(검색 조건에 lock, phantom 방지) → 비싸서 `index-range lock`(next-key locking)으로 근사.
- (주의: 2PL ≠ 2PC. 2PL=serializable isolation, 2PC=atomic commit)

### Serializable Snapshot Isolation (SSI)
- 2008년, snapshot isolation + serialization conflict 탐지. `optimistic concurrency control`(차단 대신 진행 후 commit 시 검증, contention 낮을 때 유리).
- '결정이 outdated premise에 기반했는가' 탐지: (1) stale MVCC read 탐지(무시한 write가 commit됨), (2) prior read에 영향 주는 write 탐지(tripwire, 차단 안 함).
- 장점: lock 대기 없음(예측 가능 latency), 단일 core 한계 없음(FoundationDB는 분산). PostgreSQL serializable, CockroachDB, FoundationDB. abort율이 성능 좌우 → read/write transaction은 짧아야.

---

## 4. Distributed Transactions
> 여러 node에 걸친 transaction. concurrency control은 단일 node와 유사하나 atomicity가 새 난제.

### Atomic Commit & 2PC
- 단일 node는 commit record를 디스크에 쓰는 단일 device가 atomicity 결정. 분산은 일부 commit·일부 abort 위험 → `atomic commitment problem`.
- `2PC`(two-phase commit): coordinator(transaction manager)가 phase1에서 prepare 전송 → 모두 yes면 phase2에서 commit, 하나라도 no면 abort.
- 'system of promises': 참가자가 yes 하면 commit 보장(abort 권리 포기), coordinator 결정은 디스크 기록 후 irrevocable(`commit point`).
- coordinator 장애: yes 투표 후 coordinator crash 시 참가자는 `in doubt`(commit/abort 모름) — 회복 대기뿐. lock 보유로 차단 위험. 3PC는 bounded delay 가정이라 실무 부적합 → consensus(Ch10)로 대체.

### Heterogeneous vs Internal
- `XA`(eXtended Architecture): 이종 기술 간 2PC 표준(C API, JTA). 문제: coordinator는 SPOF·app server 디스크 로그 의존, in-doubt 시 lock 장기 보유, lowest common denominator(deadlock 탐지·SSI 불가), heuristic decision(atomicity 위반).
- database-internal 분산 transaction(NewSQL: CockroachDB, TiDB, Spanner, FoundationDB)은 같은 software라 더 나은 protocol·consensus로 coordinator 복제·직접 통신 → XA 문제 회피. snapshot isolation·SSI 가능.
- `exactly-once`: distributed transaction 없이도 가능 — message ID를 DB 테이블에 기록(idempotence), uniqueness constraint로 중복 방지. Kafka Streams 방식(Ch12).

---

## Summary (핵심 정리)
- transaction은 concurrency 문제와 fault를 'abort 후 retry'로 단순화하는 추상화. 복잡한 access pattern일수록 고려할 error case를 크게 줄인다.
- isolation level: read committed(dirty read/write 방지) → snapshot isolation(read skew/phantom 방지, MVCC) → serializable(모든 anomaly 방지). lost update·write skew는 level별로 다르게 처리.
- serializability 3구현: serial execution(짧고 빠른 transaction·sharding), 2PL(전통적이나 성능 저하), SSI(optimistic, 차단 없음·확장 가능).
- 분산 atomicity는 2PC. database-internal은 잘 작동하나 XA(이종)는 coordinator SPOF·lock 보유·lowest common denominator 문제. idempotence로 atomic commit 없이 exactly-once 달성 가능.
- 다음 연결: Ch9에서 분산 시스템에서 잘못될 수 있는 것들(unreliable network/clock/process pause)을 다룸.