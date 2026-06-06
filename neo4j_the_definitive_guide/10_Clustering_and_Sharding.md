# 10. Clustering and Sharding

## 챕터 개요 (3줄 요약)

- Neo4j의 고가용성(high availability)을 위한 클러스터링이 Raft 합의 프로토콜과 `M = 2F + 1` 내결함성 공식에 기반함을 설명한다.
- 읽기 확장을 위한 세컨더리(secondary) 서버, 인과적 일관성(causal consistency, 북마크), 그리고 "1+1 클러스터"가 왜 위험한지를 다룬다.
- 멀티데이터베이스 클러스터, 네트워크 지연(latency)의 영향, 그리고 컴포지트 데이터베이스를 통한 샤딩(sharding)·페더레이션(federation)으로 마무리한다.

---

## 1. Clustering for High Availability

> 클러스터링은 Raft 합의 프로토콜로 프라이머리(primary) 서버들이 트랜잭션 순서에 합의하게 하여 고가용성을 제공한다.

- Raft 프로토콜: 프라이머리들이 리더를 선출하고 트랜잭션에 투표·합의해 데이터 일관성을 보장한다.
- 내결함성(fault tolerance): `M = 2F + 1` 공식 — F개 장애를 견디려면 M개의 프라이머리가 필요하다(1개 견디려면 3개, 2개 견디려면 5개).
- 견딜 수 있는 한도를 넘어 프라이머리가 실패하면, 데이터 안전을 위해 쓰기를 중단하고 과반(majority)이 회복될 때까지 읽기 전용(read-only)이 된다.
- 세컨더리(secondaries): Raft 합의에 참여하지 않고 로그 시핑(log shipping)으로 프라이머리의 변경을 받아 재생(replay)하며, 손실되어도 가용성에 영향이 없다.

```
Fault tolerance: M = 2F + 1
  tolerate 1 failure -> 3 primaries
  tolerate 2 failures -> 5 primaries
Lose quorum -> writes halt, cluster goes read-only
```

---

## 2. Deploying and Scaling

> 클러스터는 프라이머리로 배포되며, 세컨더리를 추가해 읽기 워크로드를 확장한다.

- 배포(deploying a cluster): Docker Compose로 3-서버 클러스터를 로컬 배포할 수 있고, 기본 모드는 `PRIMARY`이며 `initial.server.mode_constraint`로 변경한다.
- 세컨더리로 읽기 확장(scaling reads): 읽기 전용 세컨더리는 Raft에 참여하거나 리더 선출에 투표하지 않고, `db.cluster.catchup.pull_interval` 주기로 Raft 로그를 당겨와 동기화한다.
- 세컨더리는 백업 수행에도 활용해 프라이머리 부하를 줄인다.
- 클러스터 저하(cluster degradation): 과반을 잃으면 쓰기가 멈추므로 적절한 프라이머리 수가 중요하다.

---

## 3. Consistency and the 1+1 Pitfall

> 인과적 일관성은 클라이언트가 자신의 이전 쓰기 결과를 보장받게 하며, 1+1 클러스터는 과반이 없어 위험하다.

- 인과적 일관성(causal consistency): 요청이 다른 서버로 라우팅되어도 클라이언트가 최소한 자신의 이전 쓰기 효과를 보도록 보장하며, 북마크(bookmark, 경량 토큰)로 구현한다.
- 1+1 클러스터의 함정: 프라이머리 1 + 세컨더리 1 구성은 노드가 둘뿐이라 정족수(quorum)가 없어, 프라이머리 장애 시 진짜 고가용성을 제공하지 못하는 허상이다.
- 고가용성은 합의(consensus)를 요구하므로 최소 3개 프라이머리가 필요하다.

---

## 4. Multidatabase, Latency, Sharding

> 클러스터의 각 데이터베이스는 서로 다른 서버에 리더를 둘 수 있으며, 네트워크 지연과 샤딩 설계가 성능을 좌우한다.

- 멀티데이터베이스 클러스터(multidatabase clusters): 각 데이터베이스가 다른 서버에 자체 리더를 가져 리더십이 분산되므로, 서버 다운의 영향은 그 서버가 이끌던 DB에 따라 달라진다.
- 네트워크 지연(network latency): 동기 복제(synchronous replication)에 의존하는 클러스터에서 노드 간 낮은 지연은 쓰기 성능과 응답성에 직접적이므로 빠르고 안정적인 동기화를 보장해야 한다.
- 샤딩·페더레이션(sharding and federation): 컴포지트 데이터베이스(composite database, 명시적으로 생성·구성하는 가상 구성)로 데이터를 별도 그래프에 분산해, 주 그래프에 불필요하거나 노이즈가 될 데이터를 분리한다.

---

## Summary (핵심 정리)

- 고가용성 클러스터는 Raft 합의와 `M = 2F + 1` 내결함성 공식을 따르며, 과반을 잃으면 데이터 보호를 위해 읽기 전용이 된다.
- 읽기 확장은 Raft에 참여하지 않는 세컨더리로, 일관성은 북마크 기반 인과적 일관성으로 달성하며, 1+1 구성은 정족수 부재로 피해야 한다.
- 멀티데이터베이스로 리더십을 분산하고, 낮은 네트워크 지연을 확보하며, 컴포지트 데이터베이스로 샤딩·페더레이션을 구현한다.
