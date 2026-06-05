# 03. Modularity

## 챕터 개요 (3줄 요약)

- 모듈성(modularity)의 정의와 그것이 아키텍처 분석 도구의 토대가 되는 이유를 설명한다.
- 응집도(cohesion), 결합도(coupling), 동시성(connascence)이라는 세 가지 측정 개념을 다룬다.
- 추상도·불안정도·메인 시퀀스로부터의 거리 같은 파생 지표로 코드베이스 구조를 정량 평가한다.

---

## 1. Modularity Versus Granularity (모듈성 vs 입도)

> 모듈성은 시스템을 더 작은 조각으로 나누는 것, 입도(granularity)는 그 조각의 크기에 관한 것이다.

모듈성은 모놀리식에서 마이크로서비스로 가듯 시스템을 분해하는 일이며 받아들여야 한다. 그러나 입도가 문제를 일으킨다. 조각이 지나치게 작거나 서로 결합되면 Spaghetti Architecture, Distributed Monolith, Big Ball of Distributed Mud 같은 안티패턴이 생긴다. 핵심은 입도와 서비스·컴포넌트 간 결합 수준에 주의하는 것이다. 모듈성은 명시적 요구사항으로 잘 등장하지 않지만 지속 가능한 코드베이스에 필요한 암묵적(implicit) 아키텍처 특성이다.

---

## 2. Defining Modularity (모듈성의 정의)

> 본서에서 모듈성은 물리적이 아닌 '논리적'으로 관련 코드를 묶은 그룹(클래스·함수 등)을 뜻한다.

언어마다 package(Java), namespace(.NET) 같은 모듈화 메커니즘을 제공한다. 역사적으로 Dijkstra의 "Go To Statement Considered Harmful"(1968)이 구조적 프로그래밍을 열었고, 1980년대 중반 짧은 모듈러 언어 시대(Modula, Ada)를 거쳐 객체지향 언어로 이어졌다. 네임스페이스는 자산을 고유하게 식별하는 수단이며, Java는 패키지명을 물리 디렉터리 구조와 일치시켜 이름 충돌을 방지했다(이후 JAR 도입으로 classpath 혼란이 생기기도 함). 느슨한 분할로 생긴 결합은 추후 모놀리스 분해를 어렵게 만들 수 있다.

---

## 3. Measuring Modularity (모듈성 측정)

> 응집도, 결합도, 동시성이라는 언어 비종속적 지표로 모듈성을 분석한다.

### Cohesion (응집도)

응집도는 모듈의 구성 요소들이 얼마나 서로 관련되어 같은 모듈에 있어야 하는지를 나타낸다. 최선에서 최악 순으로 Functional(기능적), Sequential(순차적), Communicational(통신적), Procedural(절차적), Temporal(시간적), Logical(논리적), Coincidental(우연적) 응집도가 있다. 구조적 응집도 결여는 Chidamber & Kemerer의 LCOM(Lack of Cohesion in Methods) 지표로 측정하는데, 필드를 공유하지 않는 메서드 집합들의 합을 통해 우연적 결합을 노출시킨다. 다만 LCOM은 구조적 결여만 찾을 뿐 논리적 적합성은 판단하지 못한다(제2법칙: why > how).

### Coupling (결합도)

그래프 이론에 기반해 측정한다. Afferent coupling(구심 결합)은 들어오는 연결 수, Efferent coupling(원심 결합)은 나가는 연결 수다. (기억법: a가 e보다 앞 = incoming, efferent의 e = exit)

```
   ---> [ Code Artifact ] --->
 afferent  (component/class)  efferent
 (incoming)                   (outgoing)
```

### Core Metrics (핵심 파생 지표) — Robert C. Martin

Abstractness(추상도)는 추상 요소 대 전체(추상+구체) 요소의 비율이다. Instability(불안정도)는 원심 결합 / (원심+구심) 결합으로, 값이 높을수록 변경 시 깨지기 쉽다.

### Distance from the Main Sequence (메인 시퀀스로부터의 거리)

D = |A + I - 1|. 추상도와 불안정도의 이상적 관계선(main sequence)에 가까울수록 균형 잡힌 클래스다. 우상단으로 멀어지면 'Zone of Uselessness'(너무 추상적이라 쓰기 어려움), 좌하단으로 멀어지면 'Zone of Pain'(구현 과다로 깨지기 쉬움)에 빠진다.

```
 Abstractness
   1 |\  Zone of Uselessness
     | \
     |  \  (main sequence: A + I = 1)
     |   \
   0 |____\___ Zone of Pain
     0          1  Instability
```

지표는 본질적 복잡성(essential)과 우발적 복잡성(accidental)을 구분하지 못하므로 해석이 필요하다.

### Connascence (동시성/공변성)

Meilir Page-Jones가 정의한, 결합을 더 정밀하게 기술하는 언어다. 두 컴포넌트는 한쪽 변경이 다른 쪽 수정을 요구할 때 connascent하다. Static connascence(정적): Name, Type, Meaning(=Convention), Position, Algorithm. Dynamic connascence(동적): Execution(실행 순서), Timing(타이밍/경쟁 조건), Values(함께 변해야 하는 값), Identity(동일 엔티티 참조). 속성으로 Strength(강도), Locality(지역성), Degree(차수)가 있다. 지침: 정적을 동적보다 선호하고, 강한 형태를 약한 형태로 리팩터링하며(Rule of Degree), 거리가 멀수록 약한 결합을 사용한다(Rule of Locality). 캡슐화 경계 내부 connascence는 최대화, 경계를 넘는 것은 최소화한다.

---

## 4. From Modules to Components (모듈에서 컴포넌트로)

> 대부분의 아키텍트는 모듈을 컴포넌트라 부르며, 이는 아키텍처의 핵심 빌딩 블록이다.

문제 도메인에서 컴포넌트를 도출하는 방법은 8장에서, 그 전에 아키텍처 특성과 범위를 다룬다.

---

## Summary (핵심 정리)

- 모듈성(분해)은 받아들이되 입도(크기)에 주의해 결합 안티패턴을 피해야 한다.
- 응집도(LCOM), 결합도(afferent/efferent), 파생 지표(추상도·불안정도·D)로 구조를 정량화한다.
- 동시성(connascence)은 결합을 정밀하게 기술하는 언어로, 강한 결합을 약하게·지역적으로 관리한다.
