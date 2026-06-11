# 03. Beyond the Basics with Helm

## 챕터 개요 (3줄 요약)
- Helm install/upgrade의 5단계 처리 과정과 debugging 도구(--dry-run, helm template)를 다룬다.
- release 정보 조회(release record, helm list/get)와 history·rollback 메커니즘을 설명한다.
- install/upgrade의 고급 플래그(--generate-name, --create-namespace, --install, --wait/--atomic, --force/--cleanup-on-fail)를 다룬다.

---

## 1. Templating & dry runs
> Helm 설치는 chart 로드→values 파싱→template 실행→YAML 파싱→Kubernetes 전송 5단계이며, --dry-run과 helm template로 디버깅한다.

```
helm install mysite bitnami/drupal --dry-run   # 4단계까지 + release 정보 출력, K8s 미전송
helm template mysite bitnami/drupal            # 렌더링만(K8s 미접속, 항상 install처럼)
```
- 값 우선순위: --set > -f/--values > chart values.yaml.
- --dry-run: 디버깅용, non-YAML 정보 혼합·K8s 접속·cluster별 출력. 
- helm template: K8s 미접속, 일관된 출력, CI 적합. CRD 미접근(컴파일된 built-in kind만). --validate로 검증 가능. --post-renderer로 외부 도구(Kustomize) 연동.

---

## 2. Learning about a release
> release record는 Kubernetes Secret(`sh.helm.release.v1.<name>.vN`)으로 저장되며, helm list/get으로 상태·상세를 조회한다.

- 기본 10개 revision 보존. status: pending-install/deployed/pending-upgrade/superseded/pending-rollback/uninstalling/uninstalled/failed.
- `helm list`: 요약(name/namespace/revision/status/chart/app). 실패도 revision 증가.
- `helm get` 서브: notes / values(--revision, --all) / manifest / hooks / all. `helm inspect values <chart>`는 chart 기본값.
- helm get manifest(template 산출물) vs kubectl get(현재 상태) 비교로 디버깅.

---

## 3. History & rollbacks
> helm history로 release 이력을 보고, helm rollback으로 이전 release manifest를 재제출해 복원한다.

```
helm history wordpress
helm rollback wordpress 2     # 새 revision 생성, 이전 manifest 재제출(스냅샷 복원 아님)
helm uninstall wordpress --keep-history   # 보존 시 uninstall 후에도 rollback 가능
```
- rollback은 새 revision 추가, 대상 revision은 deployed로. 수동 편집 리소스는 3-way diff로 충돌 가능 → hand-edit 비권장.
- --keep-history 없이 uninstall하면 history·rollback 불가.

---

## 4. Deep dive into installs & upgrades
> 이름·namespace·success 기준·재시작을 제어하는 고급 플래그들을 제공한다.

- **--generate-name** / **--name-template "foo-{{ randAlpha 9 | lower }}"**: 고유 이름 자동 생성(CI용).
- **--create-namespace**: namespace 자동 생성(기본은 미생성 — global name 보안 고려). --delete-namespace 없음(kubectl delete ns 사용).
- **helm upgrade --install**: 없으면 install, 있으면 upgrade(CI용, 이름 충돌 주의).
- **--wait**: pod Running까지 대기해야 성공(--timeout). **--atomic**: 실패 시 자동 rollback. CI에선 긴 timeout 권장.
- **--force**: pod 관리 리소스 삭제·재생성(downtime 발생, 신중히). **--cleanup-on-fail**: 실패 시 새로 만든 객체 삭제.

---

## Summary (핵심 정리)
- Helm 설치는 chart→values→template→YAML→K8s 5단계이며, --dry-run(디버깅)과 helm template(렌더링·CI)로 검증한다.
- release record는 Secret으로 저장되고 helm list/get/history로 상태·상세·이력을 조회한다.
- helm rollback은 새 revision을 만들어 이전 manifest를 재제출하며, --keep-history로 uninstall 후에도 가능하다.
- 고급 플래그: --generate-name/--create-namespace/--install(CI), --wait/--atomic(성공 기준), --force/--cleanup-on-fail(upgrade 동작).
