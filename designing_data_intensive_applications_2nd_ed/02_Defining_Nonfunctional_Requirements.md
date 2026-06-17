# 02. Defining Nonfunctional Requirements

> Keywords: functional vs nonfunctional requirements, fan-out, materialized view, response time vs throughput, percentiles(p50/p95/p99/p999), tail latency, SLO/SLA, fault vs failure, fault tolerance, chaos engineering, scalability, shared-memory/disk/nothing, maintainability(operability/simplicity/evolvability)

## 챕터 개요 (3줄 요약)
- functional requirement(무엇을 하는가) 외에 performance·reliability·scalability·maintainability 같은 nonfunctional requirement가 똑같이 중요하다.
- social network home timeline case study로 fan-out·materialized view·percentile 등 대규모 시스템의 핵심 개념을 구체화한다.
- reliability는 fault(부분 고장)를 견뎌 failure(전체 중단)를 막는 fault tolerance로 달성하며, scalability·maintainability는 설계 원칙으로 관리한다.

---

## 1. Case Study: Social Network Home Timelines
> 읽기 비용이 큰 timeline query를 write 시점에 미리 계산(materialize)해 read를 빠르게 만드는 trade-off.

### Pull vs Push (fan-out)
- naive: home timeline을 매 요청 SQL JOIN(posts×follows×users)으로 조회 + polling → 초당 수억 lookup으로 폭발.
- 개선: post 발생 시 모든 follower의 home timeline에 미리 삽입(push, mailbox 모델). read는 캐시에서 즉시 제공.
- `fan-out`: 한 요청이 유발하는 downstream 요청 증가 배수(평균 follower 200 → fan-out 200).
- write 비용 증가가 trade-off지만, read 폭발보다 훨씬 저렴. spike 시 timeline delivery는 enqueue로 지연 허용.

### Materialization & 극단 케이스
- `materialized view`: query 결과를 미리 계산·갱신한 timeline cache. read↑ write↑.
- 과도한 팔로잉 user: timeline write 일부 drop(샘플만 노출) 허용.
- celebrity(수백만 follower): drop 불가 → 별도 저장 후 read 시 merge(hybrid). celebrity 처리는 별도 infra 필요.

---

## 2. Describing Performance
> performance는 response time(사용자 체감)과 throughput(자원·비용 결정)의 두 metric으로 본다.

### Response time vs Throughput
- `response time`: 요청~응답까지 경과 시간(사용자 관점, 모든 지연 포함).
- `throughput`: 초당 요청수/데이터량. hardware당 최대치 존재.
- load 증가 시 queueing으로 response time이 급증(capacity 근접 시 sharp). CPU가 이전 요청 처리 중이라 대기.
- `metastable failure`: 과부하 시 timeout→재전송(retry storm)으로 악순환, load 줄여도 reboot 전까지 회복 불가.
- 대응: exponential backoff+jitter, circuit breaker, token bucket, load shedding(선제 거절), backpressure.

### Latency 용어
- response time = service time(실제 처리) + queueing delay + network latency.
- `head-of-line blocking`: 소수 느린 요청이 뒤 요청을 막음 → response time은 client 측에서 측정해야 함.

### Percentiles
- mean은 typical을 잘 못 보여줌 → percentile 사용. median=p50.
- `tail latency`(p95/p99/p999): 가장 느린 요청이 사용자 경험·핵심 고객(데이터 많은 valuable customer)에 직접 영향. Amazon은 p999 기준 관리, p9999는 비용 대비 효과 낮아 제외.
- `tail latency amplification`: 한 end-user 요청이 여러 backend 호출 → 하나만 느려도 전체가 느림.
- `SLO`(목표)/`SLA`(계약, 미달 시 환불 등)에 percentile 사용. percentile 평균은 수학적으로 무의미 → histogram을 더해야 함(HdrHistogram, t-digest, DDSketch).

---

## 3. Reliability and Fault Tolerance
> reliability = 무언가 잘못돼도 계속 올바르게 동작. fault(부분)와 failure(전체)를 구분한다.

### Fault vs Failure / Fault Tolerance
- `fault`: 시스템 일부가 오작동(disk 고장, machine crash, 외부 service 장애).
- `failure`: 시스템 전체가 service 제공 중단(SLO 미달). fault tolerance = fault에도 service 지속.
- `SPOF`(single point of failure): fault가 곧 failure로 번지는 부분.
- `fault injection`/`chaos engineering`: 일부러 fault를 주입해 fault-tolerance 기제를 상시 검증(많은 치명 버그는 error handling 부실에서 옴). 단 security처럼 cure 불가한 건 prevention 우선.

### Hardware faults
- HDD 연 2~5%, SSD 연 0.5~1%(uncorrectable error 연 1회 수준), CPU 약 1/1000이 가끔 오답(silent corruption), RAM bit flip(ECC도 1%+), datacenter 전체 손실(화재/홍수/solar storm) 가능.
- 대응: redundancy(RAID, dual PSU, generator). 단 fault는 상관관계 있음. cloud는 개별 machine 신뢰성보다 software 레벨 HA(availability zone, rolling upgrade) 지향.

### Software faults
- 동일 software가 여러 node에 → 고도로 correlated(leap second 버그, SSD 32768시간 버그 등). cascading failure, runaway process(자원 고갈).
- 빠른 해결책 없음: 가정·상호작용 점검, 철저한 테스트, process isolation, crash-restart, retry storm 같은 feedback loop 회피, 모니터링.

### Humans and Reliability
- 설정 변경(config)이 outage 최대 원인(hardware는 10~25%). 'human error'는 원인이 아니라 sociotechnical system 증상 — blame은 비생산적.
- 대응: 테스트, config rollback, gradual rollout, observability, 잘 설계된 interface, `blameless postmortem`(처벌 없이 학습).
- reliability 중요성: Post Office Horizon scandal(버그로 수백명 부당 유죄) — 'computer는 올바르다'는 법적 가정의 위험.

---

## 4. Scalability
> scalability = load 증가에 대처하는 능력. 1차원 라벨이 아니라 '어떻게 증가하면 어떻게 대응하는가'의 질문.

### Understanding Load
- 먼저 현재 load(throughput, peak, read/write 비율, cache hit rate, user당 데이터 수)를 명확히 이해해야 'load 2배면?' 논의 가능.
- `linear scalability`: 자원 2배로 load 2배 처리(이상적). 보통 비용은 초선형으로 증가.
- startup 초기엔 미래 scale 걱정이 premature optimization — 단순·유연 우선.

### Shared-Memory / Shared-Disk / Shared-Nothing
- `vertical scaling`(scale up): 더 강한 단일 machine. shared-memory(같은 RAM). 비용 초선형, 병목.
- `shared-disk`: 독립 CPU/RAM, 공유 disk array(NAS/SAN). lock·contention이 확장성 제한.
- `shared-nothing`(scale out, horizontal): 각 node가 자체 CPU/RAM/disk, network로 협조. linear scale 잠재력·price/perf·fault tolerance↑, 단 explicit sharding(Ch7)·분산 복잡성(Ch9).
- cloud native는 storage/compute 분리로 shared-disk의 확장 문제 회피(전용 API).

```
Shared-Memory:  [ CPU+CPU | shared RAM ]  (scale up)
Shared-Disk:    [ node ][ node ] --net-- [ shared disk array (NAS/SAN) ]
Shared-Nothing: [ node: CPU+RAM+disk ] --net-- [ node: CPU+RAM+disk ]  (scale out)
```

### Principles for Scalability
- 'magic scaling sauce'는 없음 — 아키텍처는 application 특화(100k×1kB vs 3/min×2GB는 throughput 같아도 전혀 다름).
- 매 order of magnitude마다 재설계 필요. 1자릿수 이상 미리 계획 말 것.
- 원칙: 독립적으로 동작하는 작은 component로 분해(microservices, sharding, stream processing, shared-nothing) + 불필요한 복잡성 회피(단일 머신으로 되면 그게 낫다).

---

## 5. Maintainability
> 소프트웨어 비용 대부분은 초기 개발이 아니라 ongoing maintenance. operability·simplicity·evolvability로 설계.

### Operability
- 운영을 쉽게: monitoring/observability 지원, 개별 machine 의존 회피, 좋은 문서·운영 모델, 좋은 default+override, 적절한 self-healing, 예측 가능한 동작. 'good ops는 나쁜 software를 보완할 수 있지만 그 역은 어렵다.'

### Simplicity
- 복잡성(big ball of mud)은 유지비·버그 위험↑. `essential`(문제 본질) vs `accidental`(도구 한계) complexity 구분(경계는 도구 발전에 따라 이동).
- 최고의 도구는 `abstraction`: 구현 detail을 깔끔한 façade 뒤로 숨김(고급 언어, SQL, transaction, index, event log).

### Evolvability
- 요구사항은 끊임없이 변함 → 변경 용이성. loosely coupled·simple 시스템이 수정 쉬움.
- `irreversibility` 최소화가 flexibility 핵심(DB 마이그레이션 시 롤백 가능하면 위험↓). Agile의 TDD/refactoring을 시스템 레벨로 확장.

---

## Summary (핵심 정리)
- nonfunctional requirement 4종: performance, reliability, scalability, maintainability — 모두 기능만큼 중요.
- performance는 percentile·throughput으로 측정하고 SLA에 사용. reliability는 fault tolerance(hardware/software/human fault 구분, blameless postmortem)로 달성.
- scalability는 'load 정의 → 증가 시 대응'이며 작은 독립 component 분해가 핵심. maintainability는 operability·simplicity·evolvability + 좋은 building block(abstraction).
- 다음 연결: 이후 챕터에서 이 building block들(transaction, index, event log, sharding, replication)을 기술적으로 상세히 다룸.