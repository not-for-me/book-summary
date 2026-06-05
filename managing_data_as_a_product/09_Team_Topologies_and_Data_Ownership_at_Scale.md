# 09. Team Topologies and Data Ownership at Scale

## 챕터 개요 (3줄 요약)

- 팀 토폴로지(Team Topologies) 프레임워크로 데이터 관리 기능의 조직 구조를 설계하는 방법을 설명한다.
- 운영 팀(데이터 제품·플랫폼·거버넌스·활성화)과 관리 팀(전략·포트폴리오·운영 위원회)의 구조와 상호작용을 다룬다.
- 데이터 소유권의 분산(decentralization) 전략과 시점·대상·방법을 제시한다.

---

## 1. Introducing Team Topologies

> 조직 아키텍처는 팀을 기본 단위로 하며, 팀 토폴로지로 형태와 기능을 함께 설계한다.

- 조직 아키텍처 3요소: 조직도(organizational chart), 팀 구조(team structure), 운영 모델(operating model).
- 팀은 작고(최대 9명) 안정적이어야 신뢰와 응집(cohesion)을 확보한다.
- 네 가지 팀 유형: 스트림 정렬(stream-aligned), 활성화(enabling), 플랫폼(platform), 복잡 서브시스템(complicated-subsystem).
- 연합 팀(federated team): 관리 위원회(management committee)와 실천 공동체(community of practice).
- 세 가지 상호작용 모드: 협업(collaboration), XaaS(X-as-a-Service), 촉진(facilitation).

### Fractal organization
- 팀 API(SIPOC: Suppliers, Inputs, Process, Outputs, Customers 다이어그램)로 팀 활동·상호작용을 문서화한다.
- VSM(Viable System Model)은 프랙탈(fractal) 조직 구조를 만들어 전략 계획·통제를 분산시켜 민첩성을 높인다.

```
   stream-aligned (primary, deliver value)
        ^  ^  ^
        |  |  |  supported by:
   enabling (facilitation) / platform (XaaS) / complicated-subsystem
```

---

## 2. Defining operational teams

> 데이터 관리 기능의 네 핵심 운영 역량은 각각 적합한 팀 유형으로 구현된다.

- 데이터 제품 팀(data product team): 스트림 정렬 팀으로 비즈니스 도메인에 정렬되며 생애주기 전체를 자율 관리한다.
- 주로 XaaS 모드로 상호작용하며 OHS(Open Host Service)·PL(Published Language) 패턴으로 재사용성을 높인다.
- 각 팀에 데이터 제품 오너(owner)와 기술 리더(technical leader)를 두며 매트릭스 구조는 지양한다.
- 플랫폼 팀(platform team): XOps 플랫폼을 XaaS로 제공하며 IT 부서에 중앙집중되고, 인제스천(ingestion) 팀도 여기 속한다.
- 거버넌스 팀(governance team): 정책 수립 역량을 연합/하이브리드 팀으로 구현하며 정책을 제품처럼 관리한다.
- 활성화 팀(enabling team): 촉진(facilitation) 모드로 제품·플랫폼·거버넌스 팀 간 피드백을 연결한다.

---

## 3. Defining management teams

> 데이터 관리 기능은 정체성 정의, 전략 도출, 실행 계획·통제의 관리 역량을 위원회로 구현한다.

- 데이터 전략 위원회(data strategy committee): VSM System 5(정체성)+4(전략)을 구현하며 비전·목표·MoS·적합성 함수를 연/반기 단위로 정의한다.
- 데이터 포트폴리오 관리 위원회(data portfolio management committee): System 3(통제)+3*(감사)를 구현하며 비전을 이니셔티브로 번역하고 자원을 배분한다.
- 모든 조직 요구는 포트폴리오 위원회를 거쳐야 전략 계획 일관성이 유지된다.
- 운영 관리 위원회(operations management committee): System 2(조정)를 구현하며 다팀 활동을 조정(PMO 역할).
- Team Topologies·VSM·EDGE가 시너지로 모듈·적응형·빠른 흐름 아키텍처를 정의한다.

---

## 4. Evaluating decentralization strategies

> 데이터 제품 관리는 소유권 분산을 강제하지 않으며, 분산은 모듈화가 가능케 하는 선택지이다.

- 분산 비권장 경우: 중앙집중으로 충분한 규모, IT 외부 후원 부족, 낮은 데이터 성숙도, 도메인 동의 가능성 낮음, 자원/시간 제약.
- 분산 대상은 보강 수준으로 분류: 원시 데이터(raw, 소스 정렬)와 변환 데이터(transformed, 소비자 정렬).
- 변환 데이터만 분산: 가장 보편적이며 IT가 소싱, 도메인이 소비자 정렬 제품을 개발한다.
- 원시 데이터만 분산: 제품 중심 디지털 기업에서 도메인이 자체 앱의 소스 정렬 제품을 개발할 때 흔하다.
- 분산은 데이터 제품 관리와 함께 또는 이후에 진행하며, 부트스트랩 종료~확장 시작 사이가 적기이다.
- 변화에 개방적이고 교차기능 팀 구성 자원이 있는 도메인부터 점진적으로 온보딩한다.

---

## Summary (핵심 정리)

- Team Topologies의 4가지 팀 유형과 3가지 상호작용으로 데이터 관리 기능 조직을 설계했다.
- 운영 역량(데이터 제품·플랫폼·거버넌스·활성화)과 관리 역량(전략·포트폴리오·운영 위원회) 팀을 분석했다.
- 소유권 분산은 선택적이며, 무엇을·언제·어떻게 분산할지 비판적·점진적으로 접근할 것을 권장했다.
