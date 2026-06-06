# 11. Observability

## 챕터 개요 (3줄 요약)

- Neo4j 관측성(observability)을 로그(logs)와 메트릭(metrics)이라는 두 축으로 다루며, 네 가지 로그 유형과 설정·검사·튜닝 방법을 설명한다.
- 서버 부하·Neo4j 부하·워크로드(Bolt·객체 수·처리량) 메트릭으로 시스템 상태를 실시간 파악하는 법을 소개한다.
- Grafana·Loki·Prometheus 스택으로 로그와 메트릭을 통합 시각화·질의하는 실전 구성을 보여준다.

---

## 1. Harnessing the Power of Logs

> 로그는 시스템 이면의 사건을 기록하는 조용한 이야기꾼으로, 메모리 고갈 같은 문제의 근본 원인을 진단하는 데 필수적이다.

- Neo4j 로그 4종류: Neo4j(서버 운영 전반 개요), Debug(문제 진단용 상세 정보), Security(보안 이벤트), Query(쿼리 실행 기록). 그 외 GC(가비지 컬렉션) 로그도 참고한다.
- 로그 설정: `user-logs.xml`(Neo4j 로그)와 `server-logs.xml`(나머지 로그) 두 설정 파일을 사용하며, 어펜더(appenders)와 로거(loggers)로 출력 방식·범위를 제어한다.
- 로그 검사(inspecting): Neo4j·Security·Debug·Query·GC 로그를 각각 확인해 상황을 파악한다.
- 위치는 배포 환경/OS에 따라 다르며, Docker에서는 컨테이너 명령으로 설정을 확인한다.

---

## 2. Taming the Query Log

> 쿼리 로그는 `server-logs.xml` 외에도 `neo4j.conf`의 여러 설정으로 동작이 좌우되며, 적절히 길들여야 유용하다.

- `db.logs.query.enabled`: `OFF`/`INFO`/`VERBOSE` 중 선택 — `OFF`는 비활성, `VERBOSE`는 상세 기록.
- 로그 이벤트 필터링으로 불필요한 항목을 걸러낸다.
- 메타데이터 보강(enriching metadata): 애플리케이션 식별자 등 메타데이터를 추가해 추후 질의를 쉽게 한다.
- 장시간 실행 쿼리(long-running queries) 식별: 임계 시간을 설정해 느린 쿼리를 골라낸다.

---

## 3. Unveiling the Power of Metrics

> 로그가 과거의 이야기라면, 메트릭은 실시간 스냅샷으로 추세 파악·이상 탐지·안정성 유지를 돕는다.

- 서버 부하 메트릭(server load metrics): CPU 사용률, 메모리 사용량, 여유 디스크 공간. CPU가 100%에 근접하면 증설이 필요하고, 디스크 부족은 크래시·데이터 손상 위험이 있다.
- Neo4j 부하 메트릭: Neo4j 인스턴스 자체의 자원 사용을 측정한다.
- 워크로드 메트릭(workload metrics): Bolt 메트릭(현재 Cypher를 실행 중인 연결 수), 객체 수(object count) 메트릭, 처리량(throughput) 메트릭.
- 메트릭 활성화(enabling metrics) 후 모니터링 시스템으로 수집한다.

```
Logs   = retrospective ("what happened")  -> Neo4j/Debug/Security/Query/GC
Metrics = real-time snapshot ("what's happening now")
          server load (CPU/RAM/disk) | Neo4j load | workload (Bolt/objects/throughput)
```

---

## 4. Logs and Metrics with Grafana, Loki, Prometheus

> 관측성 스택을 구성하면 로그와 메트릭을 한곳에서 시각화하고 질의할 수 있다.

- 스택 구성(observability stack): GitHub 저장소의 Docker Compose로 Neo4j와 함께 Grafana·Loki·Prometheus 등을 한 번에 띄운다.
- 메트릭 시각화: Prometheus가 수집한 메트릭을 Grafana 대시보드로 본다.
- 로그 질의(querying logs): Grafana의 Explore에서 Loki 데이터 소스를 골라 Loki 질의 언어로 로그를 단계적으로(전체 → 애플리케이션 필터 → 쿼리 시간 필터 → 결합) 검색한다 — 잘 정의된 메타데이터가 이때 빛을 발한다.
- 기타 도구: Neo4j Ops Manager(UI 기반 모니터링·관리), Neo4j Aura의 Query analyzer(쿼리 수·지연·실패 타임라인 검토).

---

## Summary (핵심 정리)

- 관측성은 로그(회고적 분석)와 메트릭(실시간 스냅샷)의 두 축으로 구성되며, Neo4j는 네 가지 로그와 다양한 메트릭을 제공한다.
- 쿼리 로그는 `neo4j.conf` 설정으로 상세도·필터·메타데이터를 조절해 느린 쿼리를 식별하는 데 활용한다.
- Grafana·Loki·Prometheus 스택이나 Neo4j Ops Manager/Aura Query analyzer로 로그와 메트릭을 통합 분석할 수 있다.
