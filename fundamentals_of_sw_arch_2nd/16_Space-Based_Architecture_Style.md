# 16. Space-Based Architecture Style

## 챕터 개요 (3줄 요약)

- 공간 기반(space-based) 아키텍처는 데이터베이스 병목을 제거해 극단적 확장성·탄력성·동시성을 다루는 스타일이다.
- 중앙 DB 대신 인메모리 복제 데이터 그리드를 사용하고, 데이터 펌프로 비동기 DB 갱신(eventual consistency)을 한다.
- 확장성·탄력성·성능이 최고(5점)이나 복잡성·테스트성·비용 면에서 트레이드오프가 크다.

---

## 1. Topology (토폴로지)

> tuple space(공유 메모리 병렬 처리) 개념에서 유래하며, 인메모리 데이터 그리드로 DB 제약을 대체한다.

주요 산출물: Processing Unit(애플리케이션 기능+인메모리 그리드), Virtualized Middleware(조정), Messaging Grid(요청·세션), Data Grid(데이터 동기화·복제), Processing Grid(다중 PU 오케스트레이션), Deployment Manager(PU 시작/종료), Data Pump(비동기 DB 전송), Data Writer(DB 갱신), Data Reader(DB 읽기).

```
   [Processing Units] (in-memory replicated grid)
          |  data pump (async)
   [Data Writer] -> [ Database ] -> [Data Reader]
   --- Virtualized Middleware (messaging/data/processing grid, deployment mgr) ---
```

---

## 2. Style Specifics (스타일 세부)

> 처리 유닛은 애플리케이션 로직과 인메모리 데이터 그리드(Hazelcast, Apache Ignite, Oracle Coherence)를 담는다.

### Caching (캐싱)

- Replicated cache(복제 캐시, 표준): 각 PU가 동기화된 인메모리 그리드 보유. 매우 빠르고 단일 장애점 없음. 단 100MB 초과 데이터나 높은 갱신율엔 부적합.
- Distributed cache(분산 캐시): 외부 중앙 캐시 서버. 높은 일관성. 단 원격 접근 지연·단일 장애점.
- Near-cache(근접 캐시, 비권장): front cache(부분)+backing cache(전체). PU 간 front cache 미동기화로 불일치.

```
  Replicated: 성능·내결함성 우선, 소형(<100MB), 정적 데이터, 낮은 갱신율
  Distributed: 일관성 우선, 대형(>500MB), 동적 데이터, 높은 갱신율
```

### 주요 컴포넌트

- Messaging Grid: 요청·세션 관리(round-robin~next-available, HAProxy/Nginx).
- Data Grid: 같은 이름 캐시 간 비동기 복제(<100ms). 멤버 리스트로 인스턴스 추적.
- Processing Grid: 다중 PU 오케스트레이션(선택적).
- Deployment Manager: 부하 따라 PU 시작/종료(탄력성 핵심, Kubernetes).
- Data Pump: 항상 비동기 메시징, eventual consistency, 보장 전달·FIFO.
- Data Writer/Reader: DB 갱신/읽기. Reader는 전체 크래시·재배포·아카이브 조회 시에만 호출. 둘이 data abstraction layer를 형성(PU와 DB 스키마 디커플링, 변환 로직).

---

## 3. Data / Cloud / Risks / Governance / Teams

> PU가 DB와 직접 상호작용하지 않아 DB 토폴로지가 매우 유연하다.

데이터 토폴로지: 리포팅·분석은 모놀리식, 도메인 분할 가능하면 도메인 기반. 클라우드: 하이브리드(PU·미들웨어는 클라우드, 물리 DB는 온프레미스) 가능 — 독특한 강점. 위험: 잦은 DB 읽기(아카이브/콜드 스타트만 허용), 데이터 동기화·일관성(데이터 펌프 병목, 손실 방지는 영속 큐+client-ack), 높은 데이터 볼륨, Data Collision(복제 지연보다 갱신율이 높을 때 충돌 — 공식: CollisionRate = N*UR^2 / (S*RL)). 거버넌스: 메모리 소비·동기화 시간·데이터 펌프 큐 깊이·DB 읽기 빈도를 적합성 함수로 추적. 팀: 기술 분할이라 기술 정렬 팀에 효과적(complicated-subsystem 팀이 data grid/충돌 처리 담당).

```
  Collision Rate = (N * UR^2) / (S * RL)
   N=instances, UR=update rate, S=cache size(rows), RL=replication latency
```

---

## 4. Style Characteristics (특성 평가)

> 탄력성·확장성·성능이 5점으로 최고, 단순성·테스트성·비용이 트레이드오프다.

인메모리 캐싱+DB 제약 제거로 수백만 동시 사용자 처리 가능. 단순성·테스트성은 낮음(캐싱·eventual consistency 복잡, 고부하 테스트는 보통 운영 환경에서). 비용 높음(복잡성·캐싱 제품 라이선스·자원). 기술 분할. quanta는 UI-PU 연관과 PU 간 동기 통신으로 결정(DB는 quantum에 미포함).

```
  Elasticity/Scalability/Performance ***** (max)
  Simplicity/Testability  * 
  Cost  high
```

---

## 5. Examples and Use Cases (예시)

> 사용자·요청 폭주가 크고 1만 동시 사용자 초과 처리량이 필요한 앱에 적합하다.

콘서트 티켓 시스템(인기 공연 발매 시 수백→수만 급증, 몇 분 만에 매진 — Deployment Manager가 발매 직전 PU 대기). 온라인 경매 시스템(예측 불가 입찰 폭주, 경매별 PU 할당으로 입찰 데이터 일관성, 비동기 데이터 펌프로 이력·분석 전송). 응답성·확장성·탄력성을 동시에 극대화하는 유일한 스타일.

---

## Summary (핵심 정리)

- 공간 기반 아키텍처는 인메모리 복제 그리드로 DB 병목을 없애 극단적 확장성·탄력성을 달성한다.
- 데이터 펌프·라이터·리더로 비동기 eventual consistency를 구현하며 데이터 충돌 관리가 핵심이다.
- 성능·확장성·탄력성이 최고지만 복잡·고비용·테스트 곤란하여 폭주성 고부하 앱에 특화된다.
