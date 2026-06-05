# 09. Foundations

## 챕터 개요 (3줄 요약)

- 아키텍처 스타일과 패턴의 차이, 그리고 기본 패턴(Big Ball of Mud, 클라이언트/서버, 3계층)을 정의한다.
- 기술적 분할 vs 도메인 분할이라는 최상위 분할(top-level partitioning)과 Conway의 법칙을 설명한다.
- 모놀리식 vs 분산 아키텍처를 구분하고, 분산 컴퓨팅의 오류(fallacies) 11가지와 팀 토폴로지를 다룬다.

---

## 1. Styles Versus Patterns (스타일 vs 패턴)

> 아키텍처 스타일은 컴포넌트 토폴로지·물리 구조·배포·통신·데이터 토폴로지를 기술하는 반면, 패턴은 맥락화된 해법을 담는다.

스타일 이름은 복잡한 요인들을 간결히 지칭한다. 스타일은 공식 위원회가 정하지 않고 진화하는 생태계에서 등장한다. 예: microservices는 DevOps·오픈소스 OS·DDD의 부상으로 생긴 '라벨'이며, 가장 작은 서비스를 만들라는 계명이 아니다.

---

## 2. Fundamental Patterns (기본 패턴)

> 계층 분리 같은 기본 패턴은 소프트웨어 역사만큼 오래됐으며 반복적으로 나타난다.

### Big Ball of Mud (진흙 공, 안티패턴)

식별 가능한 구조가 전혀 없는 아키텍처. 스파게티 코드, 무분별한 전역 정보 공유가 특징. 모든 것이 결합되어 변경 시 예측 불가한 부작용이 생기고, 배포·테스트·확장·성능 모두 나쁘다. 코드 품질·구조 거버넌스 부재로 자주 발생.

### Unitary / Client-Server (단일/클라이언트-서버)

초기엔 컴퓨터와 소프트웨어가 하나였다가 분리됐다. 2계층(client/server)은 프론트/백엔드를 분리: (1) 데스크톱+DB 서버, (2) 브라우저+웹 서버, (3) 단일 페이지 JavaScript 앱. 3계층(three-tier)은 1990년대 후반 인기로 DB·애플리케이션·프론트엔드 계층을 나눴고 CORBA·DCOM 프로토콜과 결합했다. (Java의 serialization은 3계층 전성기에 언어에 내장됐다가 스타일이 사라진 뒤에도 호환성 부담으로 남았다 — 단순 설계가 미래 대비 전략.)

---

## 3. Architecture Partitioning (아키텍처 분할)

> 최상위 분할은 기술적 분할(layered monolith)과 도메인 분할(modular monolith)로 나뉘며 근본적 스타일을 정의한다.

기술적 분할은 presentation·business·persistence 등 기술 역량으로 조직한다(MVC 패턴과 매칭, 많은 조직의 기본값). 도메인 분할은 DDD에서 영감받아 도메인/워크플로우로 조직한다(microservices 기반).

```
 Technical partitioning:   Domain partitioning:
   [ Presentation ]         [ CatalogCheckout ]
   [ Business    ]          [ Promotion       ]
   [ Persistence ]          [ Delivery        ]
   (domain smeared          (workflow contained
    across layers)           in component)
```

### Conway's Law (콘웨이의 법칙)

"시스템을 설계하는 조직은 그 조직의 의사소통 구조를 복제한 설계를 만든다." Inverse Conway Maneuver는 원하는 아키텍처를 위해 팀 구조를 함께 진화시키는 것(team topologies). 기술적 분할은 워크플로우가 계층을 가로지르고(CatalogCheckout이 모든 계층에 번짐), 도메인 분할은 변화를 더 잘 반영한다. 업계는 도메인 분할로 가는 추세다.

### Kata: Silicon Sandwiches 분할

도메인 분할은 비즈니스에 가깝고 분산 마이그레이션이 쉽지만 커스터마이징 코드가 여러 곳에 흩어진다. 기술적 분할은 커스터마이징을 명확히 분리하지만 전역 결합이 높고 데이터 분리가 어렵다.

---

## 4. Monolithic Versus Distributed (모놀리식 vs 분산)

> 모놀리식(단일 배포)과 분산(원격 프로토콜로 연결된 다중 배포)으로 분류되며, 분산은 공통 난제를 공유한다.

모놀리식: Layered(10장), Pipeline(12장), Microkernel(13장). 분산: Service-based(14장), Event-driven(15장), Space-based(16장), SOA(17장), Microservices(18장).

### Fallacies of Distributed Computing (분산 컴퓨팅의 오류)

원래 8가지(Deutsch 외, 1994) + 저자 추가 3가지:

1. 네트워크는 신뢰할 수 있다 → 아니다(타임아웃·서킷브레이커 필요).
2. 지연(latency)은 0이다 → 원격 호출은 ms 단위. 평균뿐 아니라 95~99 백분위 'long tail' 지연이 성능을 죽인다.
3. 대역폭(bandwidth)은 무한하다 → 불필요한 데이터 전송(stamp coupling)이 대역폭을 소모. 필요한 데이터만 전송하라.
4. 네트워크는 안전하다 → 모든 엔드포인트를 보호해야 해 공격 표면이 급증.
5. 토폴로지는 변하지 않는다 → 항상 변한다(네트워크 업그레이드가 지연 가정을 무효화).
6. 관리자는 한 명이다 → 대기업엔 수십 명. 조정 비용 큼.
7. 전송 비용은 0이다 → 분산은 하드웨어·게이트웨이 등으로 비용이 훨씬 크다.
8. 네트워크는 동질적이다 → 여러 벤더 혼재로 패킷 손실 가능.
9. 버저닝은 쉽다 → 버전 범위·개수·폐기 등 트레이드오프 다수.
10. 보상 업데이트(compensating updates)는 항상 작동한다 → 보상 자체가 실패할 수 있음.
11. 관찰성(observability)은 선택이다 → 분산에선 필수.

---

## 5. Team Topologies and Architecture (팀 토폴로지)

> Skelton & Pais의 팀 토폴로지는 아키텍처와 교차하는 네 가지 팀 유형을 정의한다.

Stream-aligned(특정 도메인/제품에 집중해 빠르게 가치 전달), Enabling(연구·학습으로 stream 팀 지원), Complicated-subsystem(복잡 전문 영역 담당, 인지 부하 감소), Platform(셀프서비스 API·도구를 내부 제품으로 제공).

---

## Summary (핵심 정리)

- 스타일은 토폴로지·통신·데이터 등을 기술하며 진화하는 생태계에서 등장한다.
- 최상위 분할은 기술적·도메인 분할로 나뉘고(Conway 법칙), 업계는 도메인 분할 추세다.
- 분산 아키텍처는 강력하지만 11가지 오류와 조정 비용을 감수해야 하며, 팀 토폴로지와 긴밀히 연관된다.
