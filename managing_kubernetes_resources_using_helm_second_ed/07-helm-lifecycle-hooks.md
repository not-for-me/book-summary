# 07. Helm Lifecycle Hooks

## 챕터 개요 (3줄 요약)
- Helm release lifecycle(install/upgrade/rollback/uninstall) 각 단계에 custom action을 끼우는 hook 메커니즘을 다룬다.
- hook annotation·weight·delete-policy·resource-policy 등 hook lifecycle과 cleanup 옵션을 설명한다.
- guestbook chart에 pre-upgrade(backup)·pre-rollback(restore) hook을 구현해 Redis DB를 백업·복원한다.

---

## 1. The basics of a Helm hook
> hook은 release 생애 특정 시점에 1회 실행되는 Kubernetes 리소스(주로 Job)로, `helm.sh/hook` annotation으로 일반 리소스와 구분된다.

```yaml
kind: Job
metadata:
  annotations:
    "helm.sh/hook": post-install
spec:
  template:
    spec:
      restartPolicy: Never
```
- bare pod보다 Job 권장(node 장애 시 재스케줄). 0=성공, non-0=실패.
- 기본 release timeout 5분, `--timeout`으로 조정. 실행된 hook은 unmanaged → 수동 정리 필요(또는 delete-policy).

---

## 2. Hook life cycle
> `helm.sh/hook` 값으로 실행 시점을 지정하며, weight로 동일 phase 내 순서를 제어한다.

phase: pre/post-install, pre/post-delete, pre/post-upgrade, pre/post-rollback, test(helm test).

- **helm.sh/weight**: 오름차순(작은 값 먼저). 미지정 시 kind·name 알파벳순.
- comma-list로 다중 phase 지정 가능(`pre-install,post-install`).
- install 흐름: CRD → 템플릿 검증·렌더 → pre-install hook(weight순) → 리소스 적용 → post-install hook → 결과 반환.

---

## 3. Hook cleanup
> hook은 기본적으로 uninstall 시 자동 삭제되지 않으며, hook-delete-policy·Job TTL·resource-policy로 정리·보존을 제어한다.

- **helm.sh/hook-delete-policy**: before-hook-creation(기본, 생성 전 기존 삭제), hook-succeeded(성공 시 삭제), hook-failed(실패 시 삭제). comma-list 가능.
- Job `ttlSecondsAfterFinished: 60`: 완료 N초 후 자동 삭제.
- **helm.sh/resource-policy: keep**: 일반 리소스(예 standalone PVC)를 uninstall 후에도 보존(재설치 시 이름 충돌 주의).

---

## 4. Writing hooks in Guestbook (backup/restore)
> pre-upgrade hook으로 Redis dump.rdb를 backup PVC에 복사하고, pre-rollback hook으로 backup을 master에 복원 후 replica를 재배포한다.

**pre-upgrade (templates/backup/)**:
- persistentvolumeclaim.yaml: weight 0, redis.master.persistence.enabled 조건, revision-1 기반 이름(sub 함수).
- job.yaml: weight 1, delete-policy before-hook-creation+hook-succeeded, redis-cli save → dump.rdb를 backup PVC로 복사.

**pre-rollback (templates/restore/)**:
- serviceaccount.yaml(weight 0) + rolebinding.yaml(edit ClusterRole 부여).
- job.yaml: initContainer로 backup dump.rdb를 master에 복사·reload, 이후 Redis replica StatefulSet 재시작 → backup 상태 서빙.

```
helm install guestbook chapter7/guestbook -n chapter7 --dependency-update
helm upgrade guestbook guestbook -n chapter7    # pre-upgrade: 백업 PVC 생성
helm rollback guestbook 1 -n chapter7           # pre-rollback: 복원
```
- `--no-hooks`로 hook skip 가능(install/upgrade/rollback/delete).

---

## Summary (핵심 정리)
- hook은 release lifecycle 단계마다 1회성 Kubernetes 리소스(주로 Job)를 실행해 backup/restore·secret fetch·cleanup 등을 자동화한다.
- helm.sh/hook(시점), helm.sh/weight(순서), helm.sh/hook-delete-policy(정리), resource-policy(보존)가 핵심 annotation이다.
- guestbook에 pre-upgrade backup·pre-rollback restore hook을 추가해 Redis DB를 스냅샷·복원하는 흐름을 구현했다.
- --no-hooks로 hook 건너뛰기가 가능하며, 예시는 데모용으로 production 용도는 아니다.
