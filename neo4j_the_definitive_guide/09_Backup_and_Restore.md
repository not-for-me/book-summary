# 09. Backup and Restore

## 챕터 개요 (3줄 요약)

- Neo4j의 쓰기 경로(write path), 체크포인트(checkpoint), 트랜잭션 로그(transaction log) 보존 메커니즘을 이해하는 것이 안전한 백업의 전제임을 설명한다.
- 전체(full)·증분(incremental) 백업의 동작 원리와 `neo4j-admin` 도구를 사용한 백업/복원 방법, 클라우드·원격 백업을 다룬다.
- 내구성(durability)·성능(performance)·복구 보장(recovery guarantee)을 균형 있게 고려한 프로덕션 백업 전략 설계로 마무리한다.

---

## 1. The Write Path

> 백업 전략을 세우려면 Neo4j가 데이터를 어떻게 쓰고 영속화하는지 트랜잭션 수명주기를 먼저 이해해야 한다.

- 트랜잭션은 변경이 일어날 때 시작되며, 변경은 먼저 메모리와 페이지 캐시(page cache, 저장 파일로 가기 전 중간 계층)에 적용된다.
- 선행 기록(write-ahead) 아키텍처와 메모리 기반 페이지 캐시 때문에, 저장 파일을 단순 복사하면 불완전/손상 백업이 될 수 있다.
- 체크포인트(checkpoints): 커밋된 트랜잭션이 수정한 더티 페이지(dirty page)를 페이지 캐시에서 저장 파일로 플러시(flush)해 복구 시간을 단축하는 정리 작업이다.
- 체크포인트는 모든 변경이 반영된 최신 로그 위치를 기록해, 오래된 트랜잭션 로그를 버리고 크래시 후 더 빠르게 재시작하게 한다.

---

## 2. Transaction-Log Retention

> 트랜잭션 로그는 모든 쓰기를 기록해 내구성·크래시 복구·복제·백업에 필수적이지만, 관리하지 않으면 무한히 커진다.

- 트랜잭션 로그는 모든 쓰기 연산을 기록하며 크래시 복구, 복제, 백업에 사용된다.
- 설정에 따라 디스크를 고갈시킬 수 있으므로, Neo4j는 로그를 회전(rotate)·분할하고 오래된 세그먼트를 제거/아카이브한다.
- 보존(retention) 설정이 너무 공격적이면 증분 백업에 필요한 로그 체인이 끊길 수 있어 균형이 중요하다.

---

## 3. Backups: Full and Incremental

> 올바른 백업은 단순 파일 복사가 아니라 일관성을 보장하고 다운타임을 최소화하며 Neo4j의 내부 상태 관리와 정합해야 한다.

- 전체 백업(full backup): 데이터베이스 저장 파일 전체를 백업한다. `./bin/neo4j-admin database backup --to-path=<dir> --type=FULL <db>` 명령을 사용하며 체크포인트를 유발한다.
- 증분 백업(incremental backup): 트랜잭션 로그로 마지막 백업 이후 변경분만 식별·전송해 더 빠르고 공간 효율적이다 — 단, 마지막 전체 백업부터 이어지는 완전한 백업 체인이 필요하다.
- Neo4j 내장 백업 메커니즘은 일관성 있고 복원 가능한 백업을 보장하도록 아키텍처에 맞춰 설계되었다.

```
Full backup    : store files (complete) ──► checkpoint triggered
Incremental    : tx logs since last backup ──► requires unbroken chain from last FULL
Restore        : neo4j-admin database restore  (full + incremental chain)
```

---

## 4. Restore, Cloud, Remote, and Strategy

> 신뢰성 있는 복원과 단일 장애점(single point of failure) 제거, 그리고 균형 잡힌 전략 설계가 데이터를 실제로 보호한다.

- 복원(restoring backups): ACID를 준수해도 디스크 장애·비정상 종료로 파일 손상이 생길 수 있으므로, 전체/증분 백업을 안정적으로 복원하는 도구를 제공한다.
- 클라우드 백업(cloud backups): VM/클라우드 인스턴스에서 같은 호스트에 백업을 두면 VM 손실 시 백업도 잃으므로, Amazon S3·Azure Blob·Google Cloud Storage 같은 객체 스토리지에 별도 보관한다(`neo4j-admin`이 내장 지원).
- 원격 백업과 VM 분리(remote backups and VM separation): 운영 서버에서 직접 전체 백업을 돌리면 지연이 생기므로, 별도 VM에서 원격 실행하는 것이 좋다.
- 백업 전략 설계: 내구성(일관·복원 가능), 성능(처리량 영향 최소), 복구 보장(최소 데이터 손실·다운타임)의 세 요소를 균형 있게 맞춘다.

---

## Summary (핵심 정리)

- 쓰기 경로·체크포인트·트랜잭션 로그 보존을 이해해야 일관성 있는 백업을 만들 수 있으며, 저장 파일 단순 복사는 손상 위험이 있다.
- 전체 백업과 증분 백업을 `neo4j-admin`으로 수행하되, 증분은 끊기지 않은 백업 체인이 필요하다.
- 백업은 별도 VM·클라우드 객체 스토리지에 보관해 단일 장애점을 없애고, 내구성·성능·복구 보장을 균형 있게 설계한다.
