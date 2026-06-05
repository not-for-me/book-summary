# 07. Automating Data Product Lifecycle Management

## 챕터 개요 (3줄 요약)

- XOps(eXtended Operations) 플랫폼의 데이터 생태계 내 역할과 두 가치 엔진(트랜잭션·학습)을 설명한다.
- 개발자·운영·소비자 경험을 향상시키는 핵심 역량(빌딩 블록, 사이드카, 마켓플레이스 등)을 다룬다.
- 플랫폼 구현 시 make-or-buy(자체 개발 vs 구매) 의사결정 원칙을 제시한다.

---

## 1. Understanding the XOps platform

> XOps 플랫폼은 표준화·추상화·자동화로 마찰을 줄여 선형 가치 사슬(value chain)을 가치 네트워크(value network)로 전환한다.

- 전통적 선형 모델은 공급자·생산자·소비자가 단편적 역할만 맡아 지속 불가능하다.
- XOps 플랫폼은 모든 행위자를 참여시키며, DataOps 플랫폼(개발·운영)과 엔터프라이즈 데이터 마켓플레이스(소비)를 경험 연속체(experience continuum)로 통합한다.
- 트랜잭션 엔진(transaction engine): 표준화·추상화·자동화로 거래 비용과 인지 부하(내재적·외재적)를 줄인다.
- 학습 엔진(learning engine): 본질적(germane) 인지 부하를 다루며 비즈니스 지식과 맥락 지식을 수집·전파한다.
- 세 가지 동기 동력: 목적(purpose), 자율성(autonomy), 숙련(mastery).

### Platform architecture
- 제어 평면(control plane)과 유틸리티 평면(utility plane, 어댑터)으로 구분된다.
- 유틸리티 평면 API: executor-api(명령 실행), observer-api(이벤트 알림), validator-api(정책 검증), collector-api(신호 집계), configurator-api(설정 관리).
- 표준 API 덕분에 하위 기술 변경 시 어댑터만 교체하면 되어 제어 평면·데이터 제품에 영향이 없다.

```
   Users
     |
  [ Control Plane ]  (second-order: data product experiences)
     |  standard APIs
  [ Utility Plane ]  --> executor / observer / validator / collector / configurator
     |  adapters
   External services (multi-cloud, CI/CD, catalogs, OPA, vault ...)
```

---

## 2. Boosting developer experience

> 플랫폼은 재사용 가능한 컴포넌트(빌딩 블록·사이드카)와 정책·온톨로지 개발 지원으로 개발 경험을 향상시킨다.

- 빌딩 블록(building block): 인프라·애플리케이션·인터페이스 또는 그 조합의 재사용 컴포넌트로, 고유 이름·버전·선언적 인터페이스를 갖는다.
- 참조(by reference) 사용이 권장되며, 복사(by copy)는 모니터링·지양한다.
- 모듈(module)은 빌딩 블록 조합, 블루프린트(blueprint)는 완전한 데이터 제품을 정의하는 모듈이다(예: 스트리밍용·배치용 블루프린트).
- 사이드카(sidecar): 여러 제품에 유용한 기능을 데이터 제품 런타임 내에서 실행한다.
- 어댑터 사이드카(입력·출력·제어·관측성·발견성 포트)와 유틸리티 사이드카(품질 체크, 익명화, 신원 해소 identity resolution)로 나뉜다.

### Policy & ontology development
- 거버넌스 팀은 정책을 코드로(PaC, Policy as Code) 정의해 계산 정책(computational policy)으로 자동 검증한다.
- 모델링 팀은 서브도메인·엔터프라이즈 온톨로지를 개발하고 데이터 자산과 의미적으로 연결(semantic linking)한다.

---

## 3. Boosting operational experience

> 운영 경험은 데이터 제품의 릴리스(배포 파이프라인 오케스트레이션)와 운영 제어 두 상호작용을 중심으로 한다.

- DPDS(Data Product Descriptor Specification)의 lifecycleInfo로 test·prod·deprecated 단계별 배포 태스크를 선언한다.
- 배포 전 디스크립터를 발행·검증(정적 정책)하고 레지스트리에 저장 후 환경에 배포, 배포 후 런타임 정책을 검증한다.
- 제어 포트로 설정·액션을 전달하고, 관측성 포트로 신호를 수집해 내부 상태를 모니터링한다.
- SLI(Service Level Indicator)·DLI(Data Quality Indicator)로 SLO(Service Level Objective)·SLA(Service Level Agreement)를 계산하고 위반 시 알림·인시던트 관리(IM)를 지원한다.

---

## 4. Boosting consumer experience

> 소비자 경험은 데이터 제품의 사용(마켓플레이스)과 조합(composition)을 중심으로 한다.

- 마켓플레이스: 탐색 가능한 카탈로그로 제품·팀·비즈니스 케이스·자산·온톨로지 개념의 관계를 표시한다.
- 검색은 정보 검색 기법과 의미 검색(semantic search)을 결합한다.
- 접근 요청 승인 워크플로, 사용 모니터링·과금, 협업(코멘트·이슈 제기)을 지원한다.
- 조합: 로우코드/노코드로 기존 제품을 결합해 새 데이터 제품이나 분석용 뷰를 생성하며 거버넌스 정책 준수를 돕는다.

---

## 5. Evaluating make-or-buy options

> XOps 플랫폼은 패러다임 전환을 보조하나 주도하지 못하므로, 운영 모델을 먼저 검증한 후 기술을 선택한다.

- 초기 동원(mobilization) 단계에는 완전한 플랫폼이 불필요하며 최소 실행 가능 플랫폼(thinnest viable platform)으로 시작한다.
- 확장 단계에서 검증된 운영 모델에 맞는 기술을 선택해 점진적으로 수확(harvested platform)한다.
- 시장에 완전한 솔루션은 없으나 데이터 개발 플랫폼+데이터 카탈로그+마켓플레이스를 통합할 수 있다.
- 오픈소스 Open Data Mesh Platform(DPDS 기반)은 make와 buy의 균형점을 제공한다.
- 미래 대비 원칙: 큐레이션된 경험, 표준 API 통합, 핵심 리소스(데이터 제품·빌딩 블록·온톨로지·정책)의 기술 독립적 공통 사양.

---

## Summary (핵심 정리)

- XOps 플랫폼은 트랜잭션·학습 엔진으로 데이터 제품·정책·온톨로지 생성 마찰을 줄이고 생태계를 동원한다.
- 개발·운영·소비자 경험을 지원하도록 핵심 역량을 오케스트레이션하는 논리 아키텍처를 분석했다.
- 플랫폼은 모듈 조합으로 점진적으로 구현하며, 각 모듈에 대해 make-or-buy를 평가하는 아키텍처 원칙을 따른다.
