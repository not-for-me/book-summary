# 02. Preparing a Kubernetes and Helm Environment

## 챕터 개요 (3줄 요약)
- Helm 실습을 위한 전제: Kubernetes cluster 접근(local은 minikube), kubectl, Helm CLI 설치.
- minikube/kubectl/Helm을 OS별 package manager 또는 직접 download로 설치하고 검증하는 절차.
- Helm 구성: upstream repository 추가, plugin, 환경 변수(XDG 경로·HELM_DRIVER 등), tab completion, kubeconfig 기반 authentication/RBAC.

---

## 1. Preparing a local Kubernetes environment with minikube
> minikube는 local single-node Kubernetes cluster를 VM/container 안에 띄워 실험·학습에 적합하며, VirtualBox를 driver로 설정해 사용한다.

```
choco/brew install minikube           # 설치
minikube config set driver virtualbox # driver 설정
minikube config set memory 4000       # RAM 4GB 권장
minikube start | stop | delete        # 핵심 3개 subcommand
```

- start: VM 생성 + cluster bootstrap.
- stop: 상태 disk 저장(재개 빠름).
- delete: cluster·VM 완전 제거.

---

## 2. Setting up kubectl
> kubectl은 Kubernetes API와 상호작용하는 공식 CLI로, minikube로 설치하거나 package manager·직접 download로 설치 후 PATH에 배치한다.

```
minikube kubectl -- version --client          # minikube 경유
choco/brew install kubernetes-cli             # package manager
kubectl version --client                      # 검증
```

- minikube cache의 binary를 PATH로 복사하면 standalone 호출 가능.

---

## 3. Setting up Helm
> Helm CLI는 package manager 또는 GitHub release archive에서 설치하며, `helm version`으로 검증한다.

```
choco install kubernetes-helm   # Windows
brew install helm               # macOS
tar -zxvf helm-*-linux-amd64.tar.gz && mv helm /usr/local/bin  # Linux
helm version
```

---

## 4. Configuring Helm
> repository·plugin·환경 변수·tab completion을 통해 Helm 동작을 조정한다.

**Repository (`helm repo`)**: add / list / remove / update / index.
```
helm repo add bitnami https://charts.bitnami.com
helm repo update   # 캐시된 metadata 갱신
```

**Plugin (`helm plugin`)**: install / list / uninstall / update. 예: Helm Diff, Helm Secrets, Helm Monitor, Helm Unittest.

**환경 변수 (XDG Base Directory 준수)**:
- HELM_CACHE_HOME: 다운로드 chart·index 캐시.
- HELM_CONFIG_HOME: repository URL·credential.
- HELM_DATA_HOME: plugin 저장.
- **HELM_DRIVER**: release state 저장 backend — secret(기본, Base64), configmap, memory, sql.
- HELM_NAMESPACE / KUBECONFIG: 기본 namespace·인증 파일 지정.

**Tab completion**: bash/zsh/fish에서 `source <(helm completion bash)` 등.

---

## 5. Authentication & Authorization/RBAC
> Helm은 kubeconfig(clusters/users/contexts)로 cluster에 authenticate하며, 권한은 Kubernetes RBAC role로 결정된다.

- kubeconfig 3요소: clusters(host+CA), users(인증정보), contexts(cluster+user+namespace binding).
- 구성 명령: `kubectl config set-cluster / set-credentials / set-context`, 조회 `kubectl config view`.
- minikube는 kubeconfig 자동 생성 → 별도 설정 불필요.
- RBAC role 예: cluster-admin(전체), edit(read/write, Helm 설치 가능), view(read-only, 설치 불가).
- minikube 사용자는 기본 cluster-admin. 비-minikube는 최소 edit 필요:
```
kubectl create clusterrolebinding $USER-edit --clusterrole=edit --user=$USER
```

---

## Summary (핵심 정리)
- Helm 실습 전제는 cluster(minikube) + kubectl + Helm CLI 설치이며, 각각 package manager나 직접 download로 설치·검증한다.
- minikube 핵심은 start/stop/delete, driver=virtualbox, memory 4GB 권장.
- Helm 구성은 repository(add/update), plugin, XDG 기반 환경 변수, HELM_DRIVER(state 저장 방식), tab completion으로 이뤄진다.
- 인증은 kubeconfig, 인가는 RBAC role(edit 이상이어야 설치 가능)로 처리되며 minikube는 cluster-admin 기본 제공.
