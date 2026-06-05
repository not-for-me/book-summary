# 06. Operating Data Products in Production

## 챕터 개요 (3줄 요약)

- 데이터 제품의 배포를 위한 CI/CD(Continuous Integration/Continuous Delivery) 원칙과 자동화된 배포 파이프라인 구축을 설명한다.
- 관측성·제어 포트와 계산 정책(computational policy)을 활용한 프로덕션 환경 거버넌스를 다룬다.
- 데이터 제품의 소비(검색·접근·조합)와 시간에 따른 진화(버전 관리·폐기) 방법을 제시한다.

---

## 1. Deploying data products

> 데이터 제품은 원자적·자율적으로 배포 가능해야 하며, 소프트웨어 개발의 CI/CD 관행을 데이터 세계에 적용한다.

- CI(Continuous Integration)는 컴포넌트 통합을 하루 한 번 이상 검증해 통합 문제를 조기에 발견한다.
- VCS(Version Control System, 예: Git)로 디스크립터와 모든 컴포넌트를 버전 관리하며, 피처 브랜치와 메인 브랜치를 분리한다.
- IaC(Infrastructure as Code, 예: Terraform)로 인프라를 선언적·재현 가능하게 프로비저닝한다.
- 통합 단계: 검증(validation)→빌드(build)→프로비저닝(provisioning)→배포(deployment)→테스트 실행.
- 단위 테스트(white box)와 통합 테스트(black box: fixture data + expectations)를 자동화한다.

### Continuous Delivery (CD) & deployment pipeline
- 테스트 환경은 프로덕션과 동일해야 하며, 배포 후 데이터 마이그레이션(migration)·백필(backfill) 작업을 수행한다.
- CD는 개발 버전이 항상 배포 가능 상태이도록 보장하며 DevOps 문화와 자동화에 의존한다.
- 배포 파이프라인은 DAG(Directed Acyclic Graph)로 구성되며 멱등적(idempotent)·재현 가능(reproducible)해야 한다.
- WAP(Write-Audit-Publish) 패턴(Netflix, 2017): 섀도 배포로 프로덕션에서 파이프라인을 테스트한다.

```
  validate -> build -> provision -> deploy -> test -> [prod: migrate -> backfill -> accept]
       (each stage gated; reuse artifacts; pipeline as code)
```

---

## 2. Governing data products

> 배포된 데이터 제품 인스턴스는 메타데이터 공유, 계산 정책, 관측성·제어 포트를 통해 운영 수명 내내 거버넌스된다.

- 각 인스턴스는 고유 URI(instance URI)로 주소 지정되며 발견성 포트(discoverability port)로 디스크립터를 노출한다.
- 디스크립터는 DPROD(DCAT 확장 온톨로지) 등 다른 포맷으로도 내보낼 수 있으며 사용자별로 필터링한다.
- 계산 정책: 정적 정책(static, 배포 전 디스크립터 검증)과 런타임 정책(runtime, 인스턴스화 후 검증)으로 나뉜다.
- 정책 엔진(Open Policy Agent, CUE, CEL)으로 정책을 코드로(policy as code) 선언한다.
- 정책 관리 컴포넌트: PDP(Policy Decision Point), PEP(Policy Enforcing Point), PAP(Policy Administration Point), PIP(Policy Information Point).

### Observing & controlling
- 관측성(observability)은 제품 역량, 모니터링은 플랫폼 역량이며, 센서(sensor)가 내부 상태 신호를 노출한다.
- 신호 유형: 로그(log), 트레이스(trace, span으로 구성, correlation ID로 연관), 메트릭(metric).
- OTLP(OpenTelemetry Protocol, CNCF Cloud Native Computing Foundation)를 신호 직렬화·전송에 권장한다.
- 데이터 관측성은 품질 체크(완전성·고유성·적시성·유효성·정확성·일관성)와 이상 탐지(anomaly detection, ML 기반)로 구현한다.
- 제어 포트(config control port, task control port)로 운영 상태 변경, 데이터 삭제, 접근 권한 부여 등 관리 작업을 수행한다.

---

## 3. Consuming data products

> 데이터 제품을 쉽게 사용하려면 검색성, 접근 요청 자동화, 데이터 조합 용이성의 세 영역을 다뤄야 한다.

- 셀프서브 플랫폼 내 데이터 제품 레지스트리(registry)가 디스크립터를 인덱싱해 데이터 허브·마켓플레이스 역할을 한다.
- 접근(access)은 제품/도메인 소유자가 결정하며, 권한은 task control port 호출로 부여되고 플랫폼이 추적한다.
- 조합(composition): 새 데이터 제품 구성 시 출력-입력 포트 어댑터가 데이터 전송을 자동 처리한다.
- 분석 애플리케이션·탐색에는 데이터 가상화(data virtualization) 계층이 출력 포트 데이터를 통합한다.
- 플랫폼은 소스→데이터 제품→분석 애플리케이션의 계보(lineage)와 의존성을 추적한다.

---

## 4. Evolving data products

> 데이터 제품의 진화는 API 생애주기 관리와 유사하며, 소비자 영향을 최소화하는 전략이 필요하다.

- 시맨틱 버저닝(semantic versioning): major(비호환 변경), minor(호환 신기능), patch(내부 구현 수정).
- 프로덕션 환경에서는 버전당 단일 인스턴스만 존재하며 최고 버전이 현재 버전이다.
- 폐기(deprecation): 가능하면 하위 호환성을 유지하고, 불가피하면 폐기 정책에 따라 구·신 버전을 공존시켜 점진 마이그레이션한다.
- 각 릴리스에는 변경 사항을 설명하는 상세 릴리스 노트를 첨부한다.

---

## Summary (핵심 정리)

- 자동화된 배포 파이프라인은 CI/CD의 기반으로 개발·릴리스를 효율적·재현 가능하게 만든다.
- 계산 정책으로 메타데이터를 자동 평가하고, 관측성·제어 포트로 인스턴스를 모니터링·관리한다.
- 검색·접근·조합으로 데이터 제품을 소비하며, 시맨틱 버저닝과 폐기 정책으로 진화를 관리한다.
