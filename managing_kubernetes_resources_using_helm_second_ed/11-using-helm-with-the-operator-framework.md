# 11. Using Helm with the Operator Framework

## 챕터 개요 (3줄 요약)
- Kubernetes operator(control loop + CRD/CR)로 desired state를 live state에 지속 reconcile하는 패턴을 다룬다.
- operator-sdk로 Helm 기반 Guestbook operator를 scaffold·build·deploy하고 CR로 chart를 설치한다.
- operator/CRD/CR 자체를 Helm chart로 관리하는 방법과 한계를 설명한다.

---

## 1. Understanding Kubernetes operators
> operator는 CR을 watch하며 애플리케이션 lifecycle을 관리하는 controller로, CRD로 API를 확장하고 day-2 운영 지식을 코드로 캡슐화한다.

- control loop: desired=live가 되도록 controller가 지속 감시·조정.
- CRD: K8s API 확장(새 resource type 등록, `kubectl get Guestbook`).
- Operator pattern(CoreOS 2016): 복잡·stateful 앱의 backup/restore/upgrade 등을 자동화.
- Operator Framework는 Go/Ansible/**Helm** 기반 operator 지원. Helm operator는 Helm chart를 control loop logic으로 사용.

---

## 2. Guestbook operator control loop
> Guestbook operator는 Guestbook CR의 생성/수정/삭제를 watch해 chart를 install/upgrade/uninstall한다.

- CR 생성 → helm install, CR 수정 → helm upgrade, CR 삭제 → helm uninstall.
- 개발 도구: operator-sdk + container tool(docker/podman/buildah). minikube VM을 작업 환경으로 활용.

---

## 3. Scaffold / build / deploy
> operator-sdk init·create api로 파일 구조를 생성하고, Makefile target으로 image build·push·deploy한다.

```
operator-sdk init --plugins helm --domain example.com
operator-sdk create api --group demo --version v1alpha1 --kind Guestbook --helm-chart <chart>
export IMG=ghcr.io/<USER>/guestbook-operator:1.0.0
make docker-build && make docker-push
make deploy     # CRD + operator 배포 (install/uninstall/deploy/undeploy)
```
- 구조: Dockerfile, Makefile, PROJECT, config/(crd/manager/rbac/samples...), helm-charts/, watches.yaml.
- 배포는 kustomize로 config/ manifest 적용. Guestbook chart hook(Job/PVC) 위해 RBAC에 권한 추가 필요.

---

## 4. Deploying the app via CR & state sync
> CLI의 helm install 대신 CR을 apply해 chart를 설치하며, operator가 desired↔live state를 지속 동기화한다.

```
kubectl apply -f config/samples/demo_v1alpha1_guestbook.yaml -n chapter11  # = helm install
helm list -n chapter11        # operator가 만든 release 확인
kubectl apply -f .../upgrade-example.yaml   # CR 수정 = helm upgrade (hook 동작)
kubectl delete -f .../guestbook.yaml        # = helm uninstall
```
- CR의 spec = values.yaml 값. values 파일 대신 CR에 값 작성.
- **한계**: release history 미보존 → 명시적 rollback 불가(실패 시 implicit rollback만). 
- **강점**: live 리소스 수정(예 replica patch) 시 operator가 CR 기준으로 즉시 revert → 항상 동기화.
- CR 먼저 삭제 후 operator 삭제(순서 중요, 아니면 uninstall 수동).

---

## 5. Managing operators/CRDs/CRs with Helm
> operator·CRD는 crds/ 폴더를 가진 chart로, CR은 별도 chart(templates/)로 관리할 수 있다.

- CRD는 chart의 **crds/** 폴더 → templates 이전 생성, 이미 있으면 skip. 단 templating·upgrade·rollback·delete 불가, cluster 권한 필요.
- CR은 templates/에 두면 Go templating·lifecycle·revision history 활용 가능. editor role(RoleBinding)로 사용자 권한 부여.

---

## Summary (핵심 정리)
- operator는 CR을 watch해 desired=live state를 reconcile하며, Helm operator는 Helm chart를 배포 메커니즘으로 사용한다.
- operator-sdk(init/create api)로 scaffold하고 Makefile로 build/push/deploy하며, CR apply로 install/upgrade/uninstall이 트리거된다.
- Helm operator는 release history 미보존(명시적 rollback 불가)이나 live 리소스 drift를 즉시 revert하는 강점이 있다.
- operator/CRD는 crds/ 폴더 chart로, CR은 별도 chart로 관리해 반복 가능한 설치를 구성할 수 있다.
