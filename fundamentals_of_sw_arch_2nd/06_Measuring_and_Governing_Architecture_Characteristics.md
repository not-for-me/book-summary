# 06. Measuring and Governing Architecture Characteristics

## 챕터 개요 (3줄 요약)

- 모호한 아키텍처 특성을 운영·구조·프로세스 측면에서 객관적으로 측정하는 방법을 다룬다.
- 순환 복잡도(Cyclomatic Complexity) 같은 구조 지표로 코드 품질을 정량 평가한다.
- 적합성 함수(fitness functions)로 아키텍처 거버넌스를 자동화하는 기법을 설명한다.

---

## 1. Measuring Architecture Characteristics (특성 측정)

> 특성 정의가 어려운 이유는 물리학이 아니고, 정의가 제각각이며, 너무 복합적이기 때문이다.

조직이 표준·구체적 정의에 합의하면 아키텍처에 대한 유비쿼터스 언어(ubiquitous language)가 생기고, 복합 특성을 측정 가능한 요소로 분해할 수 있다.

### Operational Measures (운영적 측정)

성능·확장성처럼 직접 측정 가능하나 뉘앙스가 많다. 평균 응답 시간만 보면 1%의 이상치(outlier)를 놓치므로 최댓값도 측정한다. 고수준 팀은 통계 모델을 세워 실시간 지표가 예측 범위를 벗어나면 경보를 울린다. 모바일 사용자를 위한 first contentful paint, first CPU idle 같은 성능 예산(performance budget)과 K-weight(페이지 다운로드 바이트 한도)도 측정한다.

### Structural Measures (구조적 측정)

#### Cyclomatic Complexity (순환 복잡도, CC)

Thomas McCabe(1976)가 만든, 코드 복잡도의 객관적 지표. 그래프 이론을 적용해 결정 지점(decision points)을 센다. 공식: CC = E - N + 2P (E=엣지/결정, N=노드/코드라인, P=연결 컴포넌트 수). 단일 함수는 CC = E - N + 2.

```
   if (c1 < 100) ...           CC for example = 3
   else if (c1+c2 > 500) ...   (3 - 2 + 2)
   else ...
```

업계 임계값은 10 미만이 허용이나, 저자들은 5 미만(응집적·잘 분해된 코드)을 선호한다. CC는 본질적(essential) 복잡성과 우발적(accidental) 복잡성을 구분 못 한다. 생성형 AI는 무차별 대입으로 우발적 복잡성을 잘 만든다. TDD(Test-Driven Development)는 부수효과로 더 작고 낮은 CC의 메서드를 만든다.

### Process Measures (프로세스 측정)

민첩성(agility) 같은 복합 특성은 테스트성·배포성으로 나뉜다. 테스트성은 코드 커버리지로, 배포성은 배포 성공률·소요시간·배포 유발 버그로 측정한다. 단 100% 커버리지여도 단언(assertion)이 부실하면 무의미하다.

---

## 2. Governance and Fitness Functions (거버넌스와 적합성 함수)

> 거버넌스(그리스어 kubernan='조종하다')는 아키텍트가 우선순위를 개발자가 준수하도록 보장하는 책임이다.

CI(Continuous Integration)→DevOps로 이어진 자동화 흐름이 아키텍처 거버넌스까지 확장됐다.

### Fitness Functions (적합성 함수)

진화 컴퓨팅의 개념을 빌려, 어떤 아키텍처 특성(혹은 조합)의 무결성을 객관적으로 평가하는 모든 메커니즘을 뜻한다. 새 프레임워크가 아니라 기존 도구(메트릭·모니터·유닛 테스트·카오스 엔지니어링)를 보는 새 관점이다.

#### Cyclic Dependencies (순환 의존성)

IDE의 자동 import로 컴포넌트 간 순환 참조가 생기면 모듈성이 손상되어 Big Ball of Mud로 향한다. 코드 리뷰는 너무 늦으므로, JDepend 같은 도구로 순환을 탐지하는 적합성 함수를 CI에 통합한다(중요하지만 긴급하지 않은 관심사 수호).

```
   Comp A <----> Comp B
       ^           |
       |           v
       +--------- Comp C    (cyclic dependency = bad)
```

#### Distance from the Main Sequence

JDepend나 ArchUnit으로 메인 시퀀스로부터 거리의 임계값을 정해 검증한다. (팁: 개발자가 목적을 이해하게 한 뒤 적용해야 하며, Ivory Tower식 부과는 피한다.)

#### 계층 거버넌스

Java의 ArchUnit, .NET의 NetArchTest로 계층 간 접근 규칙(예: Presentation이 Repository를 직접 참조 금지)을 유닛 테스트로 강제한다. 개발자가 단언 없는 테스트로 커버리지를 속이는 행위(gaming)도 방지할 수 있다.

#### Chaos Engineering (카오스 엔지니어링)

Netflix의 Chaos Monkey/Simian Army는 운영 환경에서 동작하는 적합성 함수다. Latency Monkey(지연), Chaos Kong(데이터센터 장애), Conformity/Security/Janitor Monkey(규칙 준수·보안·고아 서비스 제거)가 있다. "깨지느냐가 아니라 언제 깨지느냐"의 관점을 제공한다. 적합성 함수는 무거운 거버넌스가 아니라 체크리스트(The Checklist Manifesto)처럼 중요 원칙을 자동 검증하는 수단이다.

---

## Summary (핵심 정리)

- 모호한 특성을 운영·구조·프로세스 측정으로 객관화하고, 조직 차원의 유비쿼터스 언어로 통일한다.
- 순환 복잡도(CC)로 코드 복잡도를 정량화하되 본질/우발 복잡성 구분엔 해석이 필요하다.
- 적합성 함수로 모듈성·계층·카오스 등 아키텍처 원칙을 자동 거버넌스하며, 개발자 협업이 핵심이다.
