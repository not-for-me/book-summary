# 01. Understanding Kubernetes and Helm

## 챕터 개요 (3줄 요약)
- monolith → microservice → container 흐름과 그 운영 과제를 정리하고, Kubernetes가 container orchestration으로 이를 해결하는 방식을 설명한다.
- kubectl 기반 imperative/declarative 리소스 관리 방식과 각각의 한계(boilerplate, state drift, lifecycle 관리 어려움)를 짚는다.
- Helm을 Kubernetes package manager로 소개하고, 복잡도 추상화·revision history·templating·lifecycle hook 등 핵심 가치를 제시한다.

---

## 1. From monoliths to modern microservices
> monolith의 느린 배포 사이클을 극복하기 위해 DevOps와 함께 microservice로 전환되었고, container(Docker)가 packaging·isolation 수단으로 자리잡으면서 다수 container 운영 과제가 대두되었다.

- monolith: 단일 배포 단위 → 변경 시 전체 재검증 필요, 팀 간 조율로 배포 지연.
- microservice: 독립 개발·배포·scaling 가능, 리소스 footprint 축소.
- container: chroot/jail 계보의 isolation + portable packaging. Docker가 대중화.

---

## 2. What is Kubernetes?
> Google Borg에서 출발한 open source container orchestration 플랫폼으로, 리소스 기반 scheduling·HA·scalability·활발한 community를 제공한다.

- container orchestration: 리소스 요구(예: 2Gi memory, 1 CPU)에 맞는 node에 자동 배치.
- HA: service(load balancer)로 traffic 분산, 인스턴스 장애 시 무중단.
- scalability: horizontal(인스턴스 수 증가) / vertical(memory·CPU 증설) scaling.
- 활발한 community → Helm 같은 augmenting tool 생태계.

---

## 3. Deploying a Kubernetes application
> 모든 앱은 networking·storage·resource·availability·config·security를 고려해야 하며, Kubernetes에서는 이를 API resource로 표현한다.

주요 resource:
- **Pod**: 최소 배포 단위(1+ container).
- **Deployment**: Pod 집합 배포·관리, replica 유지.
- **StatefulSet**: sticky identity + 고유 PVC.
- **Service**: Pod replica 간 load-balance.
- **Ingress**: 외부 접근 제공.
- **ConfigMap / Secret**: config / 민감 데이터(Base64 obfuscation일 뿐 — 접근통제 필수).
- **PersistentVolumeClaim**: storage 요청.
- **Role / RoleBinding**: 권한 정의·부여.

---

## 4. Approaches to resource management
> kubectl로 imperative(create/edit/delete)와 declarative(apply -f YAML) 두 방식으로 리소스를 관리하며, declarative는 source control을 SOT로 삼아 유연하다.

```
kubectl <verb> <noun> <arguments>
kubectl create deployment my-deployment --image=busybox   # imperative
kubectl apply -f deployment.yaml                          # declarative
kubectl delete -f deployment.yaml                         # 삭제는 imperative 권장
```

- imperative: 빠르지만 모든 옵션을 다 노출하지 못함.
- declarative: YAML 직접 작성 → 세밀한 제어, 존재 여부로 create/modify 자동 추론.

---

## 5. Resource configuration challenges
> resource 종류가 많고, local/live state 동기화가 깨지기 쉬우며, lifecycle(install/upgrade/rollback) 추적이 어렵고, static YAML이 boilerplate를 양산한다.

- **다양한 resource type**: 깊은 학습 곡선.
- **state drift**: `kubectl edit/patch`로 live만 수정 → local과 불일치.
- **lifecycle 관리**: Kubernetes는 변경 history를 기본 보존하지 않아 rollback 지점 파악 곤란.
- **static file → boilerplate**: parameterize 불가, 유사 앱 간 중복 YAML 다량.

---

## 6. Helm to the rescue!
> Helm은 Kubernetes package manager로, dnf 같은 OS package manager와 유사한 install/upgrade/rollback/uninstall UX를 chart 단위로 제공한다.

```
helm install redis bitnami/redis --namespace=redis
helm upgrade redis bitnami/redis --namespace=redis
helm rollback redis 1 --namespace=redis
helm uninstall redis --namespace=redis
```

Helm 핵심 이점:
- Kubernetes resource 복잡도 추상화(기존 community chart 소비).
- **release history**: revision 단위 snapshot → 손쉬운 rollback.
- **values + templates**: declarative resource를 dynamic하게 parameterize → boilerplate 감소.
- local/live state 동기화 단순화(소수 parameter 관리).
- 리소스 배포 순서 자동화(Secret/ConfigMap → Deployment).
- **lifecycle hooks**: upgrade 시 backup, rollback 시 restore, install 전 검증 등 자동화.

---

## Summary (핵심 정리)
- monolith→microservice→container 전환이 Kubernetes orchestration 수요를 낳았고, Kubernetes는 scheduling·HA·scaling을 제공한다.
- kubectl의 imperative/declarative 관리는 state drift, lifecycle 추적 곤란, static YAML boilerplate라는 과제를 동반한다.
- Helm은 chart 기반 package manager로 이 과제들을 추상화·templating·revision history·lifecycle hook으로 해결한다.
- dnf↔Helm: install↔install, upgrade↔upgrade, downgrade↔rollback, remove↔uninstall.
