# 06. Advanced Chart Features

## 챕터 개요 (3줄 요약)
- chart dependency(버전 range, 조건부 활성화, child→parent 값 import)와 library chart를 다룬다.
- values JSON Schema 검증, lifecycle hook, chart test(helm test, Chart Testing)를 설명한다.
- chart 보안(provenance/integrity, PGP 서명·검증)과 CRD 관리 방법을 다룬다.

---

## 1. Chart dependencies
> Chart.yaml의 dependencies에 name·version range·repository로 선언하고, helm dependency update로 lock·다운로드한다.

```yaml
dependencies:
- name: booster
  version: ^1.0.0       # SemVer range: ^(major), ~(patch), *, X 등
  repository: https://.../repository/
```
```
helm dependency update .   # 최신 해석 → Chart.lock 생성, charts/에 다운로드
helm dependency build      # lock 버전으로 재구성
```
- 값 전달: parent values.yaml에 dependency 이름 섹션(`booster: { image: { tag: ... } }`). alias로 중복 구분.
- tight coupling(Chart.yaml, 함께 upgrade) vs loose coupling(독립 설치·service 연결).

---

## 2. Conditional deps & importing values
> condition/tags로 dependency를 토글하고, exports/child-parent format으로 child 값을 parent로 import한다.

- **condition**: `condition: booster.enabled` + values `booster: {enabled: false}` (단일 토글).
- **tags**: 여러 dependency를 공통 tag로 묶어 `tags: {faster: false}`로 토글.
- **import-values**: child의 exports를 가져오거나(`- types`), 미export 값을 `{child: types, parent: characters}`로 매핑(중첩은 `data.types`).

---

## 3. Library charts & values schema
> library chart(type: library)는 설치 불가한 재사용 template을 제공하고, values.schema.json(JSON Schema)으로 값을 검증한다.

- library chart: `type: library`, templates는 `_*.tpl`/`_*.yaml`의 named template. merge 유틸로 base+override 병합(`include "mylib.configmap" (list . "mychart.configmap")`).
- **values.schema.json**: install/upgrade/lint/template 시 computed values 검증(type, enum, pattern). 예 pullPolicy enum ["Always","IfNotPresent"] 위반 시 lint 에러.

---

## 4. Hooks & tests
> helm.sh/hook annotation으로 release 이벤트에 작업을 끼우고, test hook + helm test로 chart를 검증한다.

hook: pre/post-install, pre/post-delete, pre/post-upgrade, pre/post-rollback, test. weight(순서, ascending)·hook-delete-policy(before-hook-creation/hook-succeeded/hook-failed). `--no-hooks`로 skip.

```yaml
annotations:
  "helm.sh/hook": pre-install,pre-upgrade
  "helm.sh/hook-weight": "1"
  "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```
- **helm test**: `helm.sh/hook: test` pod 실행(install 후), 실패 시 nonzero exit. templates/tests/에 위치.
- **Chart Testing(ct)**: 상호배타 설정(ci/*-values.yaml), Chart.yaml schema 검증, YAML lint, version 증가 확인, 변경 chart만 테스트. CI용.

---

## 5. Security (provenance) & CRDs
> PGP 서명·hash로 provenance/integrity를 검증하고, CRD는 crds/ 폴더 또는 별도 chart로 관리한다.

- **provenance**: `.tgz.prov`(Chart.yaml + hash + PGP signature). `helm package --sign --key --keyring`로 생성, `helm verify`/`helm install --verify`로 검증(public key는 별도 채널 공유). GPG 2.1+는 keybox→PGP 포맷 export 필요.
- **CRD 관리**: ① crds/ 폴더(템플릿 불가, 다른 리소스보다 먼저 설치, upgrade/delete 안 함 → kubectl로). ② 별도 chart(templating·lifecycle 관리 가능, resource-policy: keep로 보존). CRD는 cluster-wide → 삭제 시 모든 custom resource 삭제, 권한·multitenant 주의.

---

## Summary (핵심 정리)
- dependency는 Chart.yaml에 version range로 선언하고 helm dependency update로 lock·다운로드하며, condition/tags 토글과 import-values 값 전달을 지원한다.
- library chart(type: library)는 재사용 template을, values.schema.json은 JSON Schema 값 검증을 제공한다.
- hook(helm.sh/hook, weight, delete-policy)으로 lifecycle 작업을, helm test·Chart Testing으로 chart를 검증한다.
- provenance(.prov, PGP 서명)로 출처·무결성을 검증하고, CRD는 crds/ 폴더나 별도 chart로 cluster-wide 특성에 주의해 관리한다.
