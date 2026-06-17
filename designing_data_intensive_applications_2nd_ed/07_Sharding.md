# 07. Sharding

> Keywords: sharding/partitioning, partition key, skew/hot shard/hot key, key-range sharding, hash sharding, mod N 문제, fixed number of shards, hash-range sharding, consistent hashing, rebalancing(auto vs manual), request routing/coordination service(ZooKeeper/etcd), multitenancy/cell-based, local vs global secondary index(document/term-partitioned)

## 챕터 개요 (3줄 요약)
- sharding은 단일 node가 감당 못 할 데이터량·write throughput을 여러 node로 나눠 horizontal scaling을 달성하는 도구다.
- 핵심은 데이터·query 부하를 균등 분산해 hot spot을 피하는 것 — key-range vs hash 두 방식, 그리고 rebalancing·request routing.
- secondary index도 sharding이 필요하며, local(document-partitioned) vs global(term-partitioned) 두 방식의 trade-off가 있다.

---

## 1. Pros and Cons of Sharding
> sharding은 scalability를 위한 heavyweight 해법 — 대규모에서만 가치, 복잡성이 대가.

- 각 record는 정확히 한 shard에 속함(shard = partition/range/region/tablet 등 명칭 다양). 보통 replication과 결합(각 shard가 여러 node에).
- read만 문제면 read scaling으로 충분 — sharding은 데이터량/write가 단일 머신 초과 시. shared-nothing horizontal scaling의 핵심.
- 복잡성: `partition key` 선택(같은 key는 같은 shard), shard 모르면 전 shard 검색, scheme 변경 어려움, 여러 shard 갱신은 distributed transaction(느림).
- 단일 머신 내에서도 core별 sharding(Redis, VoltDB, FoundationDB, NUMA 활용).

### Multitenancy
- SaaS의 tenant(고객)별 shard. 장점: resource/permission isolation, `cell-based architecture`(fault isolation), per-tenant backup·GDPR 삭제·data residency·gradual schema rollout.
- 도전: tenant가 단일 node보다 크면 tenant 내 sharding 필요, 소형 tenant 많으면 overhead(grouping+이동 문제), cross-tenant join 어려움.

---

## 2. Sharding of Key-Value Data
> 목표는 데이터·부하 균등 분산. `skew`(불균등)와 `hot shard`/`hot key`를 피해야 한다.

### Key-Range Sharding
- 연속 key 범위를 각 shard에 할당(백과사전 권 비유). 범위는 데이터 분포에 맞춰 비균등. Bigtable/HBase, CockroachDB, MongoDB(range) 등.
- shard 내 정렬 저장(B-tree/SSTable) → range scan 효율(예: 특정 월 센서 데이터).
- 단점: 인접 key에 write 몰리면 hot shard(예: timestamp key는 이번 달 shard에 집중). 해법 — sensor ID를 prefix로(분산되나 range query는 sensor별로).
- rebalancing: pre-splitting(초기 분할), shard가 크기/throughput 임계 초과 시 split(B-tree 상단과 유사). split은 비싼 연산(데이터 재기록, 부하 가중).

### Hash Sharding
- partition key를 hash해 분산(인접 무관할 때). 좋은 hash는 skew를 균등하게(MD5/Murmur3, cryptographic 불필요. Java hashCode는 프로세스마다 달라 부적합).
- `mod N` 문제: 노드 수 변경 시 대부분 key가 이동 → 비효율.
- `fixed number of shards`: 노드보다 많은 shard 생성(10 node, 1000 shard). 노드 추가 시 shard 통째 이동(split보다 저렴). 단 초기 개수 추정 필요(틀리면 비싼 resharding).
- `hash-range sharding`: 각 shard가 hash 값의 range 담당 → 부하 시 split 가능(개수 적응). range query는 비효율하나 복합 key의 2번째 column부터는 가능. YugabyteDB/DynamoDB, Cassandra/ScyllaDB(노드당 다수 range).
- `consistent hashing`: 노드 변경 시 최소 key 이동(Karger, rendezvous, jump hashing). 'consistent'는 replica/ACID consistency와 무관.

```
mod N:           node 수 변경 -> 대부분 key 이동 (나쁨)
fixed shards:    shard 통째 이동, key->shard 매핑 불변 (좋음)
hash-range:      shard가 hash range 담당, 필요 시 split (적응적)
```

### Hot spot 완화
- consistent hashing도 key 균등 ≠ 부하 균등(celebrity 등). 해법 — hot key를 전용 shard/머신, app 레벨로 key에 random 접미사(write 분산, 단 read는 전 분할 key 읽어야 함). 시간에 따라 변하는 부하·read/write hot 구분. cloud는 자동 `heat management`/`adaptive capacity`.

### Rebalancing: Auto vs Manual
- 자동: 운영 부담↓·autoscale(DynamoDB). 단 예측 불가·비싼 연산·네트워크 과부하·자동 장애감지와 결합 시 cascading failure 위험.
- manual/middle ground(Couchbase/Riak는 제안 후 승인): 느리나 운영 surprise 방지, 알려진 이벤트(Cyber Monday) 사전 대비.

---

## 3. Request Routing
> key를 어느 node에 보낼지 결정(service discovery 유사, 단 특정 shard replica만 처리 가능).

- 3 방식: (1) 아무 node 접속 후 forwarding, (2) routing tier(shard-aware LB), (3) shard-aware client 직접 접속.
- 핵심 문제: shard→node 할당 결정(coordinator, split-brain 방지), 변경 전파, cutover 처리.
- `coordination service`: ZooKeeper/etcd(consensus로 fault-tolerant)가 권위 매핑 유지, 변경 시 routing tier에 알림. HBase/SolrCloud(ZooKeeper), MongoDB(config server+mongos), Kafka/YugabyteDB/TiDB/ScyllaDB(내장 Raft). Riak은 gossip(약한 consistency, split-brain 허용).
- analytical DB는 단일 shard가 아니라 다수 shard 병렬 aggregate/join(Ch11).

---

## 4. Sharding and Secondary Indexes
> secondary index도 sharding 필요 — local vs global.

### Local Secondary Index (document-partitioned)
- 각 shard가 자기 record만 인덱싱. write는 해당 shard만 갱신(간단). 읽기는 partition key 알면 해당 shard, 모르면 전 shard query+combine(`scatter/gather`) → tail latency amplification, throughput 확장성 제약.
- MongoDB, Riak, Cassandra, Elasticsearch, SolrCloud, VoltDB.

### Global Secondary Index (term-partitioned)
- 전 shard 데이터를 covering하되 indexed value(term)로 별도 sharding. 단일 조건 query는 한 shard만 읽음. 단 실제 record는 여러 shard, 다중 조건은 postings list 교집합(네트워크 비용), write는 여러 index shard 갱신(distributed transaction 또는 async → stale).
- CockroachDB, TiDB, YugabyteDB; DynamoDB는 local·global 모두(global은 async, stale 가능).

```
Local  index: write 1 shard / read all shards (scatter-gather)
Global index: write N shards / read 1 shard for postings list
```

---

## Summary (핵심 정리)
- sharding은 단일 머신 한계를 넘는 데이터를 균등 분산해 hot spot을 피하고, 노드 추가/제거 시 rebalancing한다.
- 두 방식: key-range(정렬, range query 가능, 인접 write hot spot 위험, split으로 rebalance) vs hash(순서 파괴로 range query 비효율하나 균등, fixed shards 또는 consistent hashing). 복합 key 첫 부분을 partition key로 쓰면 나머지로 range query 가능.
- request routing은 coordination service(ZooKeeper/etcd, consensus)로 shard→node 매핑 관리. secondary index는 local(write 1·read all) vs global(write N·read 1) trade-off.
- 다음 연결: 여러 shard에 걸친 write의 부분 실패 문제 → Ch8 Transactions.