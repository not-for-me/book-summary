# 02. Using Helm

## 챕터 개요 (3줄 요약)
- helm CLI 설치·구성 방법(prebuilt binary, package manager, get script, source build)과 Kubernetes cluster 연결을 다룬다.
- chart repository 추가·검색과 install/list/upgrade/uninstall의 핵심 workflow를 실습한다(Bitnami Drupal 예시).
- installation vs release 개념, 설치 시 설정 주입(--values/--set), Helm의 release 정보 저장 방식을 설명한다.

---

## 1. Installing & configuring the helm client
> helm은 Go로 컴파일된 단일 binary로, prebuilt 다운로드·package manager·get script·source build로 설치하며 SemVer를 따른다.

```
brew/choco/snap install helm           # package manager
curl get-helm-3 | bash                 # get script
```
- SemVer X.Y.Z: major(호환성 깨짐), minor(기능 추가), patch(버그 수정). alpha/beta/rc는 pre-release. Helm 3 사용.
- cluster 연결: `$KUBECONFIG` 또는 `$HOME/.kube/config` 자동 인식(kubectl과 동일). kubectl로 credential 관리 권장.

---

## 2. Adding & searching a chart repository
> chart repository는 Helm 명세를 따르는 네트워크 파일 집합으로, helm repo add로 추가하고 helm search repo로 검색한다.

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo list
helm search repo drupal --versions   # name/label/description 검색, 버전 목록
```
- Helm 3는 기본 repo 없음 → Artifact Hub에서 찾아 추가. OCI registry 저장은 실험적(ch7).
- chart version(chart 버전) ≠ app version(패키징된 앱 버전). Helm은 chart version으로 버전 결정.

---

## 3. Installing a package & configuration
> install은 installation 이름 + chart가 필요하며, 같은 chart를 (namespace별로) 여러 번 설치할 수 있고 --values/--set으로 설정한다.

```
helm install mysite bitnami/drupal --values values.yaml
helm install mysite bitnami/drupal --set drupalUsername=admin
```
- installation = chart의 특정 인스턴스. Helm 3에선 이름이 namespace-scoped(다른 namespace면 동일 이름 가능).
- 설정: **--values**(YAML, 재현·VC 저장 용이, 권장) / **--set**(dotted notation `mariadb.db.name=...`). 민감 값은 노출 주의.

---

## 4. List / upgrade / uninstall
> upgrade는 chart 버전·설정을 변경하며 매 release마다 설정이 새로 적용되고, uninstall은 release 기록까지 삭제한다.

```
helm list --all-namespaces
helm upgrade mysite bitnami/drupal --values values.yaml   # 매번 values 제공 권장
helm upgrade mysite bitnami/drupal --version 6.2.22        # 버전 고정
helm uninstall mysite [--keep-history]
```
- **설정은 release마다 새로 적용** → upgrade 시 values 미제공하면 기본값으로 reset. 매번 동일 --values 권장.
- **--reuse-values**: 마지막 값 재사용(단 --set/--values와 혼용 비권장).
- upgrade는 변경된 최소 리소스만 적용(restart는 kubectl로).

---

## 5. How Helm stores release info
> Helm 3는 release 기록을 기본적으로 Kubernetes Secret(`sh.helm.release.v1.<name>.vN`)으로 저장한다.

- 각 revision마다 secret 1개. uninstall 시 앱 리소스 + release 기록 모두 삭제(Helm 2는 history 기본 보존, Helm 3는 기본 삭제).
- `--keep-history`로 기록 보존(rollback ch3에서 활용).

---

## Summary (핵심 정리)
- helm은 Go 단일 binary로 package manager·get script·source로 설치하며 kubeconfig로 cluster에 연결한다.
- 핵심 workflow: repo add → search → install → list → upgrade → uninstall. installation은 namespace-scoped 인스턴스, release는 변경마다 생기는 버전.
- 설정은 --values(권장)/--set으로 주입하며, upgrade 시 release마다 새로 적용되므로 매번 동일 values를 제공해야 한다.
- Helm 3는 release 기록을 Secret으로 저장하고 uninstall 시 기본 삭제하며, --keep-history로 보존할 수 있다.
