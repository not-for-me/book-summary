# 19. Choosing the Appropriate Architecture Style

## 챕터 개요 (3줄 요약)

- 아키텍처 스타일 선택은 "It depends" — 도메인·특성·전략 등 맥락에 대한 트레이드오프 분석의 정점이다.
- 아키텍처 패션 변화의 동인과, 스타일 선택 시 고려할 결정 기준을 제시한다.
- 모놀리식(Silicon Sandwiches)과 분산(Going Going Gone) 사례로 실제 선택 과정을 보여준다.

---

## 1. Shifting "Fashion" in Architecture (아키텍처 패션 변화)

> 업계의 스타일 선호는 여러 요인으로 시간에 따라 바뀐다.

동인: 과거 관찰(과거 고통에서 새 스타일 등장 — 재사용의 부정적 트레이드오프 재고), 생태계 변화(예: Kubernetes 등장), 새 역량(Docker 컨테이너 같은 패러다임 전환), 가속(변화가 점점 빨라짐 — 생성형 AI), 도메인 변화, 기술 변화, 외부 요인(라이선스 비용 등). 아키텍트는 트렌드를 이해하고 언제 따르고 언제 예외를 둘지 판단해야 한다.

---

## 2. Decision Criteria (결정 기준)

> 아키텍트는 도메인과 구조적 요소(아키텍처 특성)를 설계하며, 충분한 지식을 갖춘 뒤 스타일을 정한다.

고려 요인: 도메인 이해(운영 특성에 영향), 구조에 영향 주는 아키텍처 특성(스타척트는 도메인보다 특성을 비교 — 본서 별점 차트가 특성 중심인 이유), 데이터 아키텍처, 클라우드 배포, 조직 요인(벤더 비용·M&A), 프로세스·팀·운영 성숙도(예: Agile 미숙 시 microservices 곤란).

### Domain/Architecture Isomorphism (도메인/아키텍처 동형성)

아키텍처의 일반적 '형태'(컴포넌트 의존 구조). 일부 도메인은 토폴로지와 잘 맞는다: 커스터마이징↔microkernel, 다수 이산 연산(유전체 분석)↔space-based. 일부는 부적합: 고확장성↔대형 모놀리스(부적합), 강한 의미적 결합(다단계 보험 양식)↔microservices(부적합, service-based가 나음).

### 핵심 결정사항

```
  1. Monolith vs Distributed?
     - 단일 특성 세트면 모놀리스, 다른 세트면 분산 (quantum 개념 활용)
  2. Where should data live?
     - 모놀리스=단일 DB, 분산=어느 서비스가 영속화하나
  3. Synchronous vs Asynchronous communication?
     - TIP: 기본은 동기, 필요할 때만 비동기
```

산출물: 아키텍처 토폴로지, ADR(Architectural Decision Records), 적합성 함수.

---

## 3. Monolith Case Study: Silicon Sandwiches

> 단일 quantum으로 충분하다고 분석되어, 단순한 모놀리스가 매력적이다.

### Modular Monolith

도메인 중심 컴포넌트 + 단일 관계형 DB + 단일 웹 UI(모바일 고려). 커스터마이징은 도메인 설계로 처리 — Override 엔드포인트를 두고 모든 도메인 컴포넌트가 참조(적합성 함수로 검증). 추후 분산 전환 대비해 DB 자산도 도메인별 분리 권장.

### Microkernel

코어(도메인 컴포넌트+단일 DB) + 커스터마이징 플러그인(공통/지역, 각자 데이터로 디커플링). BFF(Backends for Frontends) 패턴으로 API 계층을 thin 마이크로커널 어댑터로 — iOS 등 기기별 포맷 변환. 두 설계 모두 극단적 성능·탄력성 불필요해 동기 통신 가능.

---

## 4. Distributed Case Study: Going, Going, Gone (GGG)

> 역할별로 다른 특성이 필요하고 야심찬 확장성·탄력성·성능 목표가 있어 microservices가 최적이다.

저수준 EDA와 microservices가 후보였으나, 운영 특성 변이 지원이 나은 microservices 선택. 컴포넌트를 서비스로 매핑: Bid Capture, Bid Streamer(읽기 전용 고성능), Bid Tracker(두 스트림 통합), Auctioneer Capture(Bid Capture와 특성 달라 분리), Auction Session, Payment, Video Capture/Streamer. 동기·비동기 혼용 — Payment가 500ms당 1건만 처리하므로 메시지 큐로 신뢰성 확보. 최종 5개 quanta(Payment, Auctioneer, Bidder, Bidder Streams, Bid Tracker). "정답"이 아니라 '최소 악' 트레이드오프.

---

## Summary (핵심 정리)

- 스타일 선택은 맥락 의존적이며 도메인·특성·데이터·클라우드·조직·팀 성숙도를 종합한다.
- 모놀리스 vs 분산, 데이터 위치, 동기 vs 비동기(기본 동기)를 핵심으로 결정한다.
- 동형성으로 도메인-토폴로지 적합성을 보고, 산출물로 토폴로지·ADR·적합성 함수를 만든다.
