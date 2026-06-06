# 02. Importing (Much) More Data

## 챕터 개요 (3줄 요약)

- 수십만~수백만 행 규모의 대용량 데이터를 Neo4j에 효율적으로 적재하기 위한 트랜잭션·배치·병렬·오프라인 기법을 다룬다.
- `LOAD CSV ... CALL { ... } IN TRANSACTIONS`, 클라이언트 드라이버의 `UNWIND` 배치, 그리고 admin import(오프라인)까지 단계별로 비교한다.
- 병렬 쓰기(parallel write) 시 발생하는 힙 메모리 경합과 잠금(locking) 문제, 그리고 추가 수집 도구(CDC, Kafka Connect, JDBC)를 소개한다.

---

## 1. Database Transactions

> 트랜잭션은 일련의 연산을 "전부 성공 또는 전부 실패"하는 하나의 단위로 묶어 데이터 무결성을 보장한다.

- 트랜잭션은 원자성(atomicity)을 보장하며, 제약(constraint) 위반 시 전체가 롤백(rollback)되어 부분 변경이 적용되지 않는다.
- 유일성(uniqueness) 같은 검사는 흔히 커밋(commit) 시점까지 지연되어, 상태 전환 중 일시적 위반을 허용한다.
- 예: `HAS_TRACK` 관계의 `position` 속성을 정수로 강제하는 타입 제약(`IS TYPED INTEGER`)을 만들 수 있다.
- 제약을 위반하는 쓰기는 거부되며 트랜잭션이 중단(abort)된다.
- 노드 키(NODE KEY) 제약은 적재 시 중복 방지와 인덱싱을 동시에 제공한다.

### The CALL IN TRANSACTIONS operation
- `LOAD CSV`를 `CALL { ... } IN TRANSACTIONS`로 감싸면 큰 파일을 여러 개의 작은 트랜잭션으로 나눠 커밋한다(기본 1,000행 단위).
- 이를 통해 힙 메모리 부족(spinning wheel of death)을 피하며 대용량을 적재한다.
- 실습: Neo4j 힙을 128MB로 제한하고 `testload` 데이터베이스를 만들어 동작을 검증한다.

---

## 2. Importing Data from Client Applications

> 애플리케이션에서 Neo4j 드라이버로 직접 적재할 때는 쿼리 파라미터와 `UNWIND` 배치를 활용해 성능을 높인다.

- Cypher 쿼리 파라미터(query parameters)는 동일한 쿼리 문자열로 입력값만 바꿔 실행하게 하여, 캐시된 실행 계획(execution plan)을 재사용해 성능을 최적화한다.
- Neo4j 드라이버(Python, Java 등)는 애플리케이션에서 데이터를 일괄 전송하는 표준 방법을 제공한다.
- `UNWIND`는 리스트(배치)를 행 단위로 펼쳐 한 번의 쿼리로 다량의 노드/관계를 생성한다.
- 배치 크기(batch sizing): 전체 파일을 한 배치로 보내면 `LOAD CSV` 전체 적재만큼 문제가 되므로, 적절한 크기로 주기적으로 커밋해야 한다.
- 너무 작으면 오버헤드, 너무 크면 메모리 부담이 생기므로 균형 잡힌 배치 크기가 핵심이다.

```
CSV file ──► driver reads rows ──► UNWIND batch (N rows) ──► commit
                                  └─► repeat until end of file
```

---

## 3. Parallel Writes

> 적재 속도를 높이려 여러 쿼리를 병렬 실행할 수 있으나, 메모리 경합과 잠금이라는 복잡한 고려사항이 따른다.

- 메모리 경합(competing for memory): 두 트랜잭션이 동시에 힙을 사용하면 두 작업을 동시에 수용할 만큼 힙이 커야 하며, 그렇지 않으면 트랜잭션이 거부되거나 서버가 다운될 수 있다.
- 잠금 메커니즘(locking mechanisms): 쓰기 시 일관성을 위해 잠금을 획득하며, 같은 레코드를 두 트랜잭션이 수정하면 하나는 다른 하나가 끝날 때까지 대기한다.
- 동일 노드에 대한 동시 업데이트(예: 한쪽은 삭제, 한쪽은 속성 변경)는 데드락이나 대기를 유발할 수 있다.
- 노드/관계를 동시에 생성하거나 관계를 동시에 추가할 때 충돌을 피하도록 데이터를 분할(partition)해야 한다.
- 병렬 적재는 적절한 힙 설정과 충돌 없는 데이터 분할이 전제될 때만 효과적이다.

---

## 4. Offline Import and Other Tools

> 초기 대규모 데이터셋은 트랜잭션 계층을 우회하는 admin import로 가장 빠르게 적재할 수 있다.

- admin import 기능은 CSV를 트랜잭션·잠금 계층을 건너뛰고 저장 계층(storage layer)에 직접 적재한다.
- 모든 CPU 코어와 I/O를 활용해 기계 자원을 최대로 사용하지만, 깨끗한(clean) 데이터셋에 최적화되어 있다.
- 관계형 테이블 → 그래프 모델 변환이 단순할 때 특히 유리하며, 복잡한 변환에는 부적합하다.
- 그 외 도구: 변경 데이터 캡처(CDC, Change Data Capture)로 트랜잭션 시스템의 변경을 스트리밍, Kafka Connect 플러그인으로 이벤트 기반 파이프라인 통합, `neo4j-jdbc`로 BI 도구/Java 환경 연동.

---

## Summary (핵심 정리)

- `LOAD CSV ... CALL IN TRANSACTIONS`와 드라이버 `UNWIND` 배치로 대용량 데이터를 안정적으로 적재하며, 배치 크기 조절이 성능의 핵심이다.
- admin import는 초기 대량 적재에 가장 빠르지만, 복잡한 데이터 변환에는 적합하지 않다.
- 병렬 쓰기는 힙 경합과 잠금을 이해하고 데이터를 적절히 분할해야 효과적이며, 지속적 수집에는 CDC·Kafka·JDBC 등을 활용한다.
