# 01. Introducing Helm

## 챕터 개요 (3줄 요약)
- cloud native 생태계와 Kubernetes의 역할(container, scheduler, declarative infra, reconciliation loop)을 개념적으로 정리한다.
- Helm을 Kubernetes package manager로 정의하고 그 설계 목표(zero-to-K8s, package management, security/reusability/configurability)를 제시한다.
- chart·template·release 등 Helm architecture의 핵심 용어와 동작을 high-level로 소개한다.

---

## 1. The Cloud Native Ecosystem
> cloud native는 cloud의 제약·역량에 맞춰 설계하는 사고 전환으로, monolith를 작은 microservice로 분해하고 container로 packaging한다.

- microservice: REST/JSON으로 통신하는 작은 단일 책임 service들의 분산 애플리케이션.
- container: program + 의존성 + 환경을 portable image로 packaging(VM과 달리 host kernel 공유).
- image: layer 단위 저장, registry로 push/pull. 식별자 name:tag@digest (tag 가변, digest 불변 SHA).
- 운영자는 의존성 관리에서 벗어나 network·storage·CPU 자원 할당에 집중.

---

## 2. Schedulers and Kubernetes
> 다수 container 실행·자원 관리 수요로 scheduler가 등장했고, Kubernetes는 declarative infrastructure와 reconciliation loop로 차별화된다.

- **declarative**: 절차가 아닌 desired state를 선언 → Kubernetes가 내부 절차로 변환.
- **reconciliation loop**: desired vs current state 비교 후 일치하도록 자동 조정.
- 핵심 resource: **Pod**(1+ container 단위 작업, init/sidecar), **ConfigMap/Secret**(설정/민감 데이터, volume·env로 주입), **Deployment**(동일 pod 집합·replica·rolling upgrade), **Service**(영속 network endpoint, label selector로 라우팅).
- manifest = resource를 YAML/JSON으로 직렬화한 것. resource type = API group + version + kind.

---

## 3. Helm's Goals
> Helm은 다수 Kubernetes resource를 하나로 packaging해 install/update/delete하는 도구로, 세 가지 목표를 둔다.

- **Zero to Kubernetes**: production-ready 예제를 즉시 설치·학습. download→설치 5분 이내 목표.
- **Package management**: Kubernetes를 OS로 보고 find/install/upgrade/delete 제공(Homebrew·Apt 모델). 차이점: 동일 앱 다중 설치(이름 부여), namespace 민감.
- **Security/Reusability/Configurability**: package authors에게 도구 제공.
  - security: provenance(출처·무결성 검증), SSL/TLS, dry-run/template/lint.
  - reusability: chart로 동일 manifest 반복 생성, 모든 K8s distro가 같은 package 공유.
  - configurability: 설치/upgrade 시 설정 주입(단, Helm은 package manager지 config management 도구 아님 — Puppet/Ansible/Helmfile/Flux 등은 별개).

---

## 4. Helm's Architecture
> chart(package), template(manifest + 디렉티브), values.yaml(기본 설정)로 구성되며, 설치 시 release 단위로 추적된다.

- **chart**: chart 명세를 따르는 파일·디렉토리 집합. Chart.yaml(메타), templates/(K8s manifest+디렉티브), values.yaml(기본값). unpacked(디렉토리) / packed(`mychart-1.2.3.tgz`). repository에 저장.
- 설치 흐름: chart 읽기 → values를 template에 주입해 manifest 생성 → Kubernetes 전송 → resource 생성.
- **installation**: 하나의 chart 설치 인스턴스(이름 부여, 다중 가능). **release**: install/upgrade/config 변경/rollback마다 생성되는 새 버전.
- Helm 2 차이: Tiller/gRPC 제거(Helm 3). 이 책은 Helm v3 + Chart v2(Helm v2는 Chart v1, deprecated).

---

## Summary (핵심 정리)
- cloud native는 monolith를 microservice·container로 분해하며, Kubernetes는 declarative state + reconciliation loop로 이를 orchestrate한다.
- 핵심 K8s resource는 Pod·ConfigMap·Secret·Deployment·Service이며 manifest(YAML/JSON)로 선언한다.
- Helm은 다수 resource를 chart로 묶는 Kubernetes package manager로, zero-to-K8s·package management·security/reusability/configurability를 목표한다.
- chart=package, template+values로 manifest 생성, installation은 설치 인스턴스, release는 변경마다 생기는 버전이다.
