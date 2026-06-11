# 10. Automating Helm with CD and GitOps

## 챕터 개요 (3줄 요약)
- 수동 Helm CLI 호출의 한계를 넘어 CI/CD·GitOps로 Helm 배포를 자동화하는 개념을 다룬다.
- Argo CD를 설치하고 Application/ApplicationSet 리소스로 Helm chart를 선언적으로 배포한다.
- Git repository·remote chart repository·multi-environment 배포 시나리오를 실습한다.

---

## 1. Understanding CI/CD and GitOps
> CI는 자동 build/test, CD는 자동 release 과정이며, GitOps는 Git을 SOT로 desired state를 cluster live state에 지속 동기화한다.

- **CI**: commit마다 자동 build·test·scan → 회귀 조기 발견, 공통 build 환경.
- **CD**: 정의된 단계로 release 진행(승인·change control). 완전 자동화는 continuous deployment.
- **GitOps**(WeaveWorks 2017): K8s manifest를 Git에 저장, controller pattern으로 live≠desired 시 자동 적용. 도구: Argo CD, Flux.
- Helm chart도 결국 K8s 리소스 → GitOps 참여 가능.

---

## 2. Installing Argo CD
> Argo CD를 community Helm chart로 설치하고 web UI에 접근한다.

```
helm repo add argo https://argoproj.github.io/argo-helm
helm install argo argo/argo-cd --version 4.5.0 --values values.yaml -n argo
kubectl get secret argocd-initial-admin-secret -n argo -o jsonpath='{.data.password}' | base64 -d
kubectl port-forward svc/argo-argocd-server 8443:443 -n argo
```
- 구성 요소: Application Controller, ApplicationSet Controller, Redis, Repo Server, Argo CD Server(API+UI).

---

## 3. Deploying from a Git repository
> Application 리소스로 Git repo의 Helm chart 경로·values·destination·syncPolicy를 선언하면 Argo CD가 `helm template` 렌더 후 적용한다.

```yaml
kind: Application
spec:
  source:
    path: helm-charts/charts/nginx/
    repoURL: https://github.com/.../...git
    targetRevision: HEAD
    helm: { values: "..." }   # 또는 parameters(=--set), valueFiles(=--values)
  destination: { server: https://kubernetes.default.svc, namespace: chapter10 }
  syncPolicy: { automated: { prune: true, selfHeal: true } }
```
- Argo CD는 helm install이 아니라 render+apply → `helm list`엔 안 보임.
- rollback은 Git revert(GitOps) 또는 Argo CD native rollback.
- finalizer로 Application 삭제 시 렌더 리소스도 삭제. prune=삭제 동기화, selfHeal=drift 복구.

---

## 4. Deploying from a remote chart repo & multiple environments
> source에 chart 이름·version·repoURL을 지정하면 remote repo 배포가 되고, ApplicationSet으로 여러 환경에 동시 배포한다.

remote repo source:
```yaml
source: { chart: nginx, targetRevision: 9.7.6, repoURL: https://.../bitnami }
```

**ApplicationSet** (multi-env):
```yaml
spec:
  generators:
    - list: { elements: [ {env: dev}, {env: prod} ] }
  template:
    metadata: { name: nginx-{{ env }} }
    spec:
      source:
        helm:
          valueFiles: [ values/common-values.yaml, values/{{ env }}/values.yaml ]
      destination: { namespace: chapter10-{{ env }} }
```
- generator(list 등)가 parameter 생성 → `{{ env }}` placeholder로 다수 Application 동적 생성.
- valueFiles는 Git 배포에서만 사용(remote repo는 values/parameters).
- 환경별 values로 replica 수 등 차등 적용(dev 1, prod 3).

---

## Summary (핵심 정리)
- CI/CD·GitOps는 Helm 배포를 Git/chart repo 내용 기반으로 자동화·확장한다.
- Argo CD는 Application(단일)/ApplicationSet(다중)으로 chart를 render+apply하며 prune·selfHeal로 desired↔live를 동기화한다.
- Git repo·remote chart repo 모두 source로 배포 가능하며, values 주입은 helm.values/parameters/valueFiles로 한다.
- ApplicationSet generator와 placeholder(`{{ env }}`)로 dev/prod 등 다중 환경에 환경별 values를 적용해 배포한다.
