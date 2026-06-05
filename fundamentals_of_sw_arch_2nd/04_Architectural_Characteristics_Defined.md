# 04. Architectural Characteristics Defined

## 챕터 개요 (3줄 요약)

- 아키텍처 특성(architectural characteristics)의 정의와 그것을 판별하는 세 가지 기준을 설명한다.
- 운영적·구조적·클라우드·횡단(cross-cutting) 범주로 나누어 대표적 특성들을 나열한다.
- 모든 특성을 동시에 최대화할 수 없으므로 '최선'이 아닌 '최소 악(least worst)' 아키텍처를 지향한다.

---

## 1. Architectural Characteristics and System Design (특성과 시스템 설계)

> 아키텍처 특성은 문제 도메인과 독립적이면서 시스템 성공에 중요한, 구조적 설계 고려사항이다.

구조적 설계는 문제 도메인 이해(시스템의 behavior)와 필요한 역량(capabilities) 발굴이라는 두 활동으로 이뤄진다. 도메인 요구사항이 '무엇을' 할지를 정의한다면, 아키텍처 특성은 '어떻게/왜'를 정의한다. 특성으로 인정되려면 세 기준을 충족해야 한다. (1) 비도메인 설계 고려사항을 명시하고, (2) 설계의 구조적 측면에 영향을 주며, (3) 애플리케이션 성공에 결정적이거나 중요해야 한다. 특성은 요구사항에 잘 안 나오는 암묵적(implicit) 특성과 명시적(explicit) 특성으로 나뉜다. 'non-functional requirements'나 'quality attributes' 대신 'architectural characteristics'라는 용어를 선호한다.

```
   Architectural Characteristic =
     (1) non-domain design consideration
   + (2) influences structural aspect of design
   + (3) critical/important to success
```

보안은 설계(암호화·해싱)로도 구조(강화된 서비스)로도 수용 가능하지만, 확장성은 모놀리스에서 일정 한계를 넘으면 분산 스타일로 '구조'를 바꿔야만 한다.

---

## 2. Architectural Characteristics (Partially) Listed (특성 목록)

> 특성은 저수준 코드 특성부터 고수준 운영 관심사까지 폭넓은 스펙트럼에 존재하며, 보편적 표준은 없다.

### Operational (운영적 특성)

가용성(Availability), 연속성(Continuity), 성능(Performance), 복구성(Recoverability), 신뢰성/안전성(Reliability/Safety), 견고성(Robustness), 확장성(Scalability) 등. 운영·DevOps 관심사와 크게 겹친다.

### Structural (구조적 특성)

설정 가능성(Configurability), 확장성(Extensibility), 설치 용이성(Installability), 재사용성(Leverageability/Reuse), 현지화(Localization), 유지보수성(Maintainability), 이식성(Portability), 업그레이드 용이성(Upgradeability).

### Cloud (클라우드 특성)

On-demand scalability/elasticity, Zone-based availability, Region-based privacy & security(국가별 데이터 거주 법규). 2판에서는 각 아키텍처 스타일 장에 클라우드 수용 방식 절을 추가했다.

### Cross-Cutting (횡단 특성)

접근성(Accessibility), 보관성(Archivability), 인증(Authentication), 인가(Authorization), 법적 제약(Legal: GDPR, Sarbanes-Oxley), 프라이버시(Privacy), 보안(Security), 지원성(Supportability), 사용성(Usability).

ISO 표준은 Performance efficiency, Compatibility, Usability, Reliability, Security, Maintainability, Portability 등으로 분류하나 불완전하다. 저자들은 Functional suitability(기능 적합성)는 특성이 아니라 동기 요구사항이라며 목록에서 제외한다. 용어 모호성을 줄이기 위해 DDD(Domain-Driven Design)의 유비쿼터스 언어(ubiquitous language)를 권장한다.

---

## 3. Trade-Offs and Least Worst Architecture (트레이드오프와 최소 악 아키텍처)

> 특성은 비용이 들고 서로 시너지(상호작용)하므로, 모두를 최대화할 수 없고 절충해야 한다.

각 특성은 설계·구현·유지 노력과 구조적 지원을 요구한다. 또한 특성들은 헬리콥터 조종처럼 상호 연동되어, 보안 강화는 흔히 성능을 저해한다. 표준 정의 부재로 조직은 자체 목록(유비쿼터스 언어)을 만들어야 한다.

```
   TIP: Never strive for the BEST architecture;
        aim for the LEAST WORST architecture.
```

너무 많은 특성을 지원하려 하면 모든 문제를 풀려는 일반적(generic) 해법이 되어 거의 작동하지 않는다. 가능한 한 반복(iterative) 가능한 아키텍처를 설계해, 변경이 쉬우면 처음에 정답을 맞히는 부담이 줄어든다.

---

## Summary (핵심 정리)

- 아키텍처 특성은 비도메인·구조 영향·성공 결정성의 세 기준을 충족하는 시스템 역량이다.
- 운영·구조·클라우드·횡단 범주로 분류되나 보편 표준은 없어 조직별 정의가 필요하다.
- 특성은 상호 시너지하며 비용이 들므로, 최소한의 특성만 골라 '최소 악' 아키텍처를 반복적으로 추구한다.
