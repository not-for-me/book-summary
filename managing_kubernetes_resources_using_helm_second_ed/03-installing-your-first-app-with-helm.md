# 03. Installing Your First App with Helm

## 챕터 개요 (3줄 요약)
- end user 관점에서 Bitnami WordPress chart를 Artifact Hub에서 찾아 minikube cluster에 설치하는 전 과정을 다룬다.
- values file 작성, install/upgrade/rollback/uninstall 등 Helm release lifecycle 명령을 실습한다.
- release 검사(helm list/get), revision history, --set vs --values, --reuse/--reset-values 동작을 설명한다.

---

## 1. Finding a chart (Artifact Hub)
> Artifact Hub는 Helm chart·operator·plugin 등 Kubernetes artifact 중앙 검색 플랫폼으로, helm search hub로 CLI 검색, browser로 상세·repo URL을 확인한다.

```
helm search hub wordpress              # Artifact Hub 전체 검색
helm search hub wordpress --output yaml
helm repo add bitnami https://raw.githubusercontent.com/bitnami/charts/archive-full-index/bitnami
helm search repo wordpress             # 추가된 repo 내 검색
```

- Bitnami 기본 index는 6개월 retention → 특정 version 보존되는 full-index repo 사용.
- `helm show chart|readme|values|all` 로 chart metadata·README·values 확인.

---

## 2. Installing a chart
> values file로 chart 기본값을 override하고 `helm install [NAME] [CHART] --values --namespace --version`로 설치한다.

`wordpress-values.yaml`:
```yaml
wordpressUsername: helm-user
wordpressPassword: my-password
service:
  type: NodePort
```
```
helm install wordpress bitnami/wordpress --values=wordpress-values.yaml -n chapter3 --version 12.1.6
```

- release: chart로 설치된 리소스 집합 + lifecycle 추적 단위.
- 설치 출력에 status(deployed)·revision(1)·notes(접근 방법) 표시.

---

## 3. Inspecting a release
> helm list로 namespace 내 release 목록을, helm get 계열로 상세를 조회한다.

- `helm list -n chapter3`: 이름·revision·status·chart·app version.
- `helm get manifest|notes|values|hooks|all`: 생성된 K8s 리소스, notes, 적용 값 등.
- `helm get values --all`: 사용자값 + 기본값 전체.
- kubectl과 label(app.kubernetes.io/name=wordpress)로 Helm 생성 리소스 필터 가능.

---

## 4. --set vs --values
> 다수·복잡 값은 --values(YAML, SCM 저장·재현 용이) 권장, 민감 값(password)은 source control 유출 방지 위해 --set 사용.

- --set: CLI 직접 전달, 단순 값에 적합. list/map은 입력 까다로움.
- --values: YAML 파일/URL, 선언적·재현 가능.
- 관련: --set-file, --set-string.

---

## 5. Upgrade / Rollback / Uninstall
> upgrade는 값 수정·chart 갱신, rollback은 이전 revision 복원, uninstall은 리소스+history 삭제.

```
helm upgrade wordpress bitnami/wordpress --values wordpress-values.yaml -n chapter3 --version 12.1.6
helm history wordpress -n chapter3      # revision 이력
helm rollback wordpress 3 -n chapter3   # revision 3으로 복원(새 revision 추가)
helm uninstall wordpress -n chapter3
```

- **--reuse-values**(값 미제공 시 기본): 이전 release 값 재사용.
- **--reset-values**(값 1개 이상 제공 시 기본): 미지정 값은 chart 기본으로 reset → 모든 값은 values file로 관리 권장.
- revision: install/upgrade/rollback마다 생성, 기본 Secret(HELM_DRIVER)에 저장. status: superseded/deployed/failed/pending 등.
- uninstall 후에도 StatefulSet이 만든 PVC는 자동 삭제 안 됨 → 수동 `kubectl delete pvc`.

---

## Summary (핵심 정리)
- end user는 Artifact Hub에서 chart를 찾아 repo add 후 helm install로 배포하며, values file로 설정을 override한다.
- release lifecycle: install → upgrade → rollback → uninstall, 각 단계가 revision으로 history에 기록된다.
- --values는 선언적·재현 가능(권장), --set은 민감값·단순값용. 값 제공 여부에 따라 reuse/reset-values 동작이 달라진다.
- rollback은 새 revision을 추가하며 비상시에만 사용하고, StatefulSet PVC는 uninstall 시 수동 삭제가 필요하다.
