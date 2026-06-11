# 06. Understanding Helm Templates

## 챕터 개요 (3줄 요약)
- Go template 기반 Helm template engine으로 values·built-in object를 받아 Kubernetes 리소스를 동적 생성하는 방법을 다룬다.
- function·pipeline·control structure·variable·named template·CRD·post rendering 등 template 전 영역을 설명한다.
- 학습 내용을 guestbook chart에 적용해 Redis 연동 env를 추가하고 frontend를 배포한다.

---

## 1. Template basics & values
> template은 `{{ }}` action 안에서 .Values 등을 placeholder로 사용해 동적으로 YAML/JSON을 생성하며, `helm template`로 설치 없이 local 렌더링한다.

```
helm template <RELEASE> <CHART> [flags]   # 설치 없이 렌더링
{{ .Values.chapterNumber }}               # values 참조
```

---

## 2. Built-in objects
> Helm은 chart 작성에 쓰는 여러 built-in object를 제공하며 dot notation으로 접근한다.

- **.Values**: values.yaml + --set/--values.
- **.Release**: Name, Namespace, IsInstall/IsUpgrade, Revision, Service.
- **.Chart**: Chart.yaml metadata(Name/Version/AppVersion).
- **.Template**: Name/BasePath.
- **.Capabilities**: APIVersions.Has, KubeVersion 등 cluster 정보.
- **.Files**: Get/Glob/AsConfig/AsSecrets/Lines — chart 내 파일 내용 주입.
- **.** : root scope.

---

## 3. Functions & pipelines
> Helm은 Go + Sprig 라이브러리의 60+ function과 Unix식 pipeline(`|`)으로 데이터를 변환·포맷한다.

```
{{ .Values.fs.path | clean | quote }}   # pipeline
{{- toYaml .Values.annotations | nindent 4 }}
{{- tpl (.Files.Get "f.cfg") . | nindent 4 }}  # 외부 파일에 templating 적용
lookup "v1" "ConfigMap" "ns" "name"     # 실행 중 cluster 조회(install/upgrade에서만)
```

- 주요: quote, default, printf, has, list, b64enc/dec, toYaml, indent/nindent, upper/lower, date.
- lookup은 cluster 연결 필요 → helm template에선 무효.

---

## 4. Control structures & variables
> if/else·with·range로 흐름을 제어하고, `$var := ...`로 변수를 정의해 scope 밖 값 참조 등에 활용한다.

```
{{- if .Values.x.enabled }} ... {{- end }}   # boolean 함수 eq/ne/lt/and/or...
{{- with .Values.deep.props }}{{ .name }} {{ $.Chart.Name }}{{- end }}  # scope 변경, $=root
{{- range $i, $v := .Values.ports }} ... {{- end }}  # 반복
```

- false 판정: false/0/빈 문자열/nil/빈 collection.
- with/range는 scope를 바꿈 → built-in object 접근은 `$` 사용.

---

## 5. Validation, named templates, CRD, post rendering
> 입력 검증은 fail/required/values.schema.json, 코드 재사용은 named template·library chart, CRD는 crds/ 폴더, 추가 수정은 post rendering으로 처리한다.

- **검증**: `fail "msg"`(즉시 실패), `required "msg" .Values.x`(빈 값 금지), values.schema.json(JSON Schema, 자동 에러).
- **named template**: `_helpers.tpl`에 `define` → `include "name" . | indent`로 사용(template action보다 include 권장).
- **library chart**: type=library, 설치 불가, helper 제공(예 Bitnami common). dependency로 import.
- **CRD**: crds/ 폴더 → templates/ 이전 생성. templating·삭제·upgrade 불가, cluster-admin 필요.
- **post rendering**: `--post-renderer <exec>`(예 Kustomize)로 렌더 후 추가 patch. 최후 수단.

---

## 6. Updating & deploying Guestbook
> Redis 값(fullnameOverride, auth.enabled=false)과 frontend env(REDIS_LEADER/FOLLOWER_SERVICE_HOST)를 추가해 완전한 Guestbook을 배포한다.

```yaml
redis: { fullnameOverride: redis, auth: { enabled: false } }
env:
  - { name: GET_HOSTS_FROM, value: env }
  - { name: REDIS_LEADER_SERVICE_HOST, value: redis-master }
  - { name: REDIS_FOLLOWER_SERVICE_HOST, value: redis-replicas }
```
```
helm install guestbook chapter6/guestbook -n chapter6
kubectl port-forward svc/guestbook 8080:80 -n chapter6
```

---

## Summary (핵심 정리)
- template은 Go template 기반으로 .Values/.Release/.Chart 등 built-in object와 function·pipeline으로 K8s 리소스를 동적 생성한다.
- if/else·with·range로 흐름 제어, variable과 `$`로 scope 문제를 해결한다.
- 입력 검증(fail/required/values.schema.json), named template/library chart(재사용), CRD(crds/), post rendering이 핵심 고급 기능이다.
- guestbook은 Redis 값과 env를 추가해 frontend↔backend 연동을 완성, 첫 동작하는 chart를 배포했다.
