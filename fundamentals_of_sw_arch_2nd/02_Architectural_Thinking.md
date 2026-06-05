# 02. Architectural Thinking

## 챕터 개요 (3줄 요약)

- 아키텍트의 시각으로 사고하는 법, 즉 아키텍처와 설계의 차이, 기술적 폭, 트레이드오프 분석을 다룬다.
- 지식 피라미드와 20분 규칙, 개인 기술 레이더를 통해 기술적 폭(breadth)을 넓히는 방법을 제시한다.
- 비즈니스 동인 이해와, 아키텍트가 직접 코딩을 병행하며 병목이 되지 않는 균형 잡기를 설명한다.

---

## 1. Architecture Versus Design (아키텍처 vs 설계)

> 아키텍처는 시스템의 '구조'에, 설계는 시스템의 '외형'에 가깝지만, 많은 결정은 둘 사이 스펙트럼 위에 존재한다.

집을 비유하면 층수·지붕 형태 같은 전체 구조는 아키텍처, 바닥재·벽 색상 같은 내부는 설계에 해당한다. 어떤 결정이 아키텍처에 가까운지는 세 가지 기준으로 판단한다. (1) 전략적인가 전술적인가, (2) 변경/구축에 드는 노력이 큰가, (3) 트레이드오프가 얼마나 중대한가. 전략적·장기적·다수 이해관계자 관여·고비용 변경·중대한 트레이드오프일수록 아키텍처에 가깝다. Martin Fowler는 아키텍처를 "바꾸기 어려운 것(the stuff that's hard to change)"이라 표현했다.

```
 Design  <-------------------------------->  Architecture
 tactical    effort to change      strategic
 short-term  significance of       long-term
 low impact  trade-offs            high impact
```

---

## 2. Technical Breadth (기술적 폭)

> 개발자는 기술적 깊이(depth)가, 아키텍트는 기술적 폭(breadth)이 더 중요하다.

지식 피라미드는 세 층으로 나뉜다. (1) 아는 것(stuff you know), (2) 모른다는 것을 아는 것(stuff you know you don't know), (3) 모른다는 것조차 모르는 것(stuff you don't know you don't know). 개발자는 꼭대기(깊이)를 키우지만, 아키텍트는 폭이 더 중요하므로 일부 깊이를 희생해 폭을 넓혀야 한다. 한 가지 캐싱 제품 전문가보다 10개 제품의 장단점을 아는 편이 낫다. 흔한 역기능으로는 모든 분야 전문성 유지 시도와, 낡은 지식을 최신으로 착각하는 'stale expertise'가 있다.

### Frozen Caveman Antipattern (얼어붙은 원시인 안티패턴)

과거의 나쁜 경험에 사로잡혀 모든 아키텍처마다 비합리적 우려를 반복하는 아키텍트를 가리킨다. 진짜 위험과 인지된 위험을 구분하고, 현실적인 리스크 평가를 하는 것이 아키텍처적 사고의 핵심이다.

```
        /\        Stuff you know (depth)
       /  \
      /----\      Stuff you know you don't know
     /      \
    /--------\    Stuff you don't know you don't know (largest)
```

### The 20-Minute Rule (20분 규칙)

매일 최소 20분을 새로운 주제 학습에 투자한다. InfoQ, DZone, Thoughtworks Technology Radar 등이 좋은 출처다. 점심·저녁보다 이메일 확인 전 아침 시간이 가장 효과적이다.

### Developing a Personal Radar (개인 레이더 만들기)

특정 기술에 과몰입하면 메메틱 버블(echo chamber)에 갇혀 외부 평가를 놓치기 쉽다. Thoughtworks 레이더는 4개 사분면(Tools, Languages & Frameworks, Techniques, Platforms)과 4개 링(Hold, Assess, Trial, Adopt)으로 구성된다. 기술 포트폴리오를 금융 포트폴리오처럼 다각화(diversify)하라.

---

## 3. Analyzing Trade-Offs (트레이드오프 분석)

> 아키텍처는 검색하거나 LLM(Large Language Model)에게 물을 수 없는, "It depends"로 답할 수밖에 없는 트레이드오프의 영역이다.

경매 시스템 예시에서 비동기 통신에 큐(point-to-point)와 토픽(publish-subscribe) 중 선택해야 한다. 토픽은 확장성(extensibility)과 서비스 디커플링, 모니터링/스케일링 면에서 유리하다. 그러나 누구나 데이터에 접근 가능해 보안 문제가 있고(토픽은 도청이 쉬움), 동질적 계약(homogeneous contract)만 지원한다. Rich Hickey의 말처럼 프로그래머는 이점만 알지만 아키텍트는 트레이드오프까지 이해해야 한다. AMQP(Advanced Message Queuing Protocol)는 exchange와 queue 분리로 프로그래밍적 로드밸런싱을 지원한다.

```
 Topic (pub-sub):   Producer --> [Topic] --> Sub A / Sub B / Sub C
   + extensibility, decoupling, scalability
   - security (easy wiretap), homogeneous contract only

 Queue (p2p):       Producer --> [Q1][Q2][Q3] --> Consumer A/B/C
   + security, heterogeneous contracts, individual monitoring
   - more coupling, infra change when adding consumer
```

---

## 4. Understanding Business Drivers (비즈니스 동인 이해)

> 비즈니스 동인을 확장성·성능·가용성 같은 아키텍처 특성으로 번역하는 능력이 필요하다.

이는 도메인 지식과 이해관계자와의 협력 관계를 요구한다. 본서는 4·5·6·7장에 걸쳐 아키텍처 특성의 정의, 식별, 측정, 범위를 다룬다.

---

## 5. Balancing Architecture and Hands-On Coding (아키텍처와 코딩 병행)

> 아키텍트는 코딩을 계속하며 기술적 깊이를 유지하되, 병목(Bottleneck Trap)이 되지 않아야 한다.

Bottleneck Trap은 아키텍트가 시스템 핵심 경로 코드를 소유해 팀의 병목이 되는 안티패턴이다. 이를 피하려면 핵심 코드는 팀에 위임하고, 1~3 이터레이션 뒤의 작은 비즈니스 기능을 직접 구현한다. 코딩을 함께 못 할 경우엔 잦은 PoC(Proof of Concept) 작성, 기술 부채(tech debt) 해소, 버그 수정, 자동화 도구 제작(예: Java의 ArchUnit 적합성 함수), 코드 리뷰 등으로 손을 놓지 않을 수 있다.

---

## Summary (핵심 정리)

- 아키텍처는 전략성·변경 노력·트레이드오프 중대성으로 설계와 구분되며 둘은 스펙트럼이다.
- 아키텍트는 깊이보다 폭이 중요하며, 20분 규칙·개인 레이더로 폭을 넓히고 트레이드오프를 분석한다.
- 비즈니스 동인을 특성으로 번역하고, 병목을 피하면서 코딩을 병행해 기술 감각을 유지해야 한다.
