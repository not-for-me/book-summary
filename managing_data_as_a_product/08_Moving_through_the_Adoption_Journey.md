# 08. Moving through the Adoption Journey

## 챕터 개요 (3줄 요약)
- 데이터를 제품으로 관리하는 패러다임 도입은 조직 구조와 문화를 모두 바꾸는 점진적·변혁적 여정이다.
- Rogers의 혁신 확산 이론(diffusion of innovation)을 기반으로 assessment, bootstrap, expand, sustain의 4단계로 도입 여정을 정의한다.
- EDGE 운영 모델을 통해 각 단계의 필요에 맞춰 진화하는 적응형 데이터 전략(adaptive data strategy)을 수립한다.

---

## 1. Understanding adoption phases
> 새 패러다임은 하룻밤에 정착하지 않으며 점진적·단계적으로 조직 전체에 확산되어야 한다.

- Rogers는 변화 수용도에 따라 채택자를 innovators, early adopters, early majority, late majority, laggards 5가지로 분류한다.
- 채택의 누적 분포는 S자형 로지스틱 곡선(logistic curve)을 그리며, 이를 bootstrap, expand, sustain 3단계로 나눈다.
- 본격 도입 전 패러다임의 타당성을 검증하는 assessment(평가) 단계가 선행되어야 한다.
- 각 단계는 순차적으로 완료되어야 하며, 가장 어려운 부분은 단계 간 전환 지점이다.
- 패러다임 전환은 상향식(bottom-up)만으로는 어렵고, CDO/CIO 등 최고 경영진의 강력한 후원(sponsorship)이 필수다.
- 데이터-제품 패러다임은 네트워크 효과로 가치를 얻으므로 innovators/early adopters를 넘어 확산되지 못하면 실패한다.

---

## 2. Delving into the assessment phase
> 핵심 이해관계자 간 비전 정렬을 만들어 새 패러다임 도입 여부와 방법을 결정하는 워크숍 중심 단계다.

- 결과물: 패러다임에 대한 공통 이해, 가치에 대한 공감, 운영 모델 수립, expand 단계 로드맵.
- 반드시 타임박스(time-box)해야 하며, 통상 2일~2개월 범위로 제한해 효과 저하와 과잉분석(over-analysis)을 방지한다.
- 분석(analysis) 미팅은 "왜(why)" 도입하는지, 종합(synthesis) 미팅은 "어떻게(how)" 도입하는지를 다룬다.
- 분석은 kick-off → inception(현재 이슈·목표 식별, vision board) → envisioning(기회·위협 분석, RAID 매트릭스) 순으로 진행한다.
- 종합 단계에서 목표를 측정 가능한 MoS(Measures of Success)로 번역하고, lighthouse projects와 예산을 확보한다.

```
 RAID matrix: Risks | Assumptions | Issues | Dependencies
```

---

## 3. Delving into the bootstrap phase
> innovators/early adopters를 대상으로 도입 여정의 토대를 마련하고 첫 데이터 제품을 구축하는 단계다.

- 평가 단계에서 정의한 운영 모델을 실행하기 위해 관리 기능(System 2-5)의 역할과 책임을 재구성한다.
- 4가지 핵심 역량(data product development, governance policy-making, XOps platform engineering, data transformation enabling)을 운영 수준(System 1)에서 구현한다.
- 실제 비즈니스 가치를 주는 lighthouse projects를 신중히 선택하되, 너무 단순하거나 과도하게 복잡한 케이스는 피한다.
- 거버넌스 정책은 점진적으로 정의하며, 데이터 제품 정의·메타데이터·descriptor document 표준부터 시작한다.
- 가장 얇은 실행 가능 플랫폼(thinnest viable platform)을 즉시 구현하고, data product registry부터 점진적으로 기능을 추가한다.
- 내부 enabling으로 시작해 점차 외부 그룹 대상 enabling으로 전환하며 handbook(살아있는 문서)을 준비한다; bootstrap은 보통 6~18개월 소요된다.

```
 GDR = Governance Decision Records (policy documentation)
```

---

## 4. Delving into the expand phase
> early/late majority로 확산을 스케일링하는 단계로, "캐즘(chasm)" 극복이 핵심 분기점이다.

- 캐즘은 위험을 감수하는 early adopters와 검증된 솔루션을 원하는 실용주의자(majority) 사이의 단절 지점이다.
- 새 패러다임을 회사 전략에 명확히 통합하고, 조직 단위별 목표와 인센티브를 제공해야 한다.
- 중앙집중/탈중앙/하이브리드 모델에 따라 스케일링 방식이 달라지며, 동질적 그룹 단위로 점진적 온보딩한다.
- 거버넌스는 도메인 간 의미론적(semantic) 상호운용성까지 보장해야 하며, XOps 플랫폼은 building blocks·blueprints·sidecars로 개발을 단순화한다.
- 레거시 시스템은 빅뱅 마이그레이션을 피하고, pseudo-data product로 격리 후 strangler fig 패턴으로 점진 리팩토링한다.

---

## 5. Delving into the sustain phase
> 대부분 조직이 패러다임을 채택한 후 규모의 경제를 극대화하기 위해 실무를 산업화하는 단계다.

- 채택 과정이 확립되어 비가역적이 되며, 초점은 채택 확대에서 규모의 경제 활용으로 이동한다.
- 지능형 자동화(intelligent automation): AI 기법으로 데이터 제품 개발·운영 관리를 추가 자동화해 시간과 비용을 절감한다.
- 사용자 경험 향상: low-code/no-code 인터페이스로 비즈니스 사용자가 기존 데이터 제품을 독립적으로 조합하게 한다.
- 산업화와 혁신은 대립이 아닌 동전의 양면이며, 둘 사이의 균형이 지속 가능한 경쟁 우위의 핵심이다.

---

## 6. Driving the adoption with an adaptive data strategy
> EDGE 운영 모델을 통해 도입 여정 전반을 성공적으로 이끄는 애자일·분산·적응형 데이터 전략을 정의한다.

- 적응형 전략은 사전 정의된 계획이 아니라 전략 목표와 운영 행동의 정렬을 항상 보장하는 모델이다.
- EDGE는 LVT(Lean Value Tree), 가치 기반 계획 프로세스, 경량 거버넌스 프레임워크 3요소로 구성된다.
- LVT는 vision → goals → initiatives → activities로 전략을 실행으로 연결하는 계획·소통·모니터링 도구다.
- 우선순위는 value score(가치)와 impact score(투자·위험)의 비율로 결정되며 계획 주기마다 조정된다.
- 적합도 함수(fitness functions)는 MoS와 보완적으로 패러다임 채택 진척도를 측정한다.

```
 LVT: Vision -> Goals -> Initiatives -> Activities
       (root)                          (leaves)
 cadence: vision(multi-year) > goals/MoS(yearly)
          > initiatives(quarterly) > activities(monthly)
```

---

## Summary (핵심 정리)
- Rogers의 혁신 확산 이론을 바탕으로 assessment, bootstrap, expand, sustain의 4단계 도입 여정을 정의했다.
- 각 단계는 고유한 목표·도전과제·운영 및 조직 활동을 가지며, 단계 간 전환(특히 캐즘)이 가장 중요한 분기점이다.
- EDGE 운영 모델과 LVT·fitness functions를 활용해 각 단계에 맞춰 진화하는 적응형 데이터 전략을 수립할 수 있다.
