# 04. Scaffolding a New Helm Chart

## 챕터 개요 (3줄 요약)
- chart 사용자에서 chart 개발자로 전환하여 Guestbook(PHP frontend + Redis backend) chart를 처음부터 만들기 시작한다.
- Helm chart 전반에 쓰이는 YAML/JSON 포맷 기초(key-value, map, list, value type)를 정리한다.
- helm create로 chart 구조를 scaffold하고, 생성된 파일과 Chart.yaml(chart definition) 필드를 이해·수정한다.

---

## 1. Understanding the Guestbook application
> Guestbook은 PHP frontend가 메시지를 Redis(leader-follower 복제 cluster)에 저장·조회하는 간단한 데모 앱이다.

- frontend: dialog + Submit → Redis leader에 write.
- Redis: leader가 follower로 replicate, frontend replica가 follower에서 read.

---

## 2. Understanding the YAML format
> YAML은 key-value(공백·들여쓰기 기반) 포맷으로 K8s·Helm 파일 대부분을 구성하며, JSON은 그 superset 관계의 대안 포맷이다.

```yaml
name: LearnHelm          # key: value (콜론 뒤 공백 필수, tab 금지)
resources:               # map (중첩, 2-space 들여쓰기 관행)
  limits:
    cpu: 100m
servicePorts:            # list
  - 8080
  - 8443
configuration: |         # multiline string
  server.port=8443
```

- value type: string(따옴표 선택), integer(`1`) vs string(`"1"`), boolean(true/false/yes/no/on/off), list, object.
- JSON: `{}`=block, `[]`=list. YAML과 상호 변환 가능.

---

## 3. Scaffolding with helm create
> `helm create NAME`은 동작하는 기본 chart 구조(기본은 NGINX)를 생성한다.

생성 구조:
- **Chart.yaml** (필수): chart metadata.
- **templates/** (필수): Golang template → K8s 리소스 생성. (deployment.yaml, service.yaml, _helpers.tpl, NOTES.txt, tests/ 등)
- **values.yaml**: 기본값(권장).
- **charts/**: dependency chart.
- .helmignore, 그 외 선택: Chart.lock, crds/, README.md, LICENSE, values.schema.json.

---

## 4. Deploying the scaffolded chart
> local path로 chart를 설치해 publish 없이 테스트할 수 있으며, scaffold 기본은 `nginx:1.16.0` 배포다.

```
helm install guestbook ./guestbook -n chapter4
helm get manifest guestbook -n chapter4   # ServiceAccount + Service + Deployment
kubectl -n chapter4 port-forward svc/guestbook 8080:80
helm uninstall guestbook -n chapter4
```

---

## 5. Chart.yaml (chart definition)
> Chart.yaml은 chart metadata를 담는 필수 파일이며, 일부 필드는 Artifact Hub 표시에 사용된다.

필수: **apiVersion**(v2=Helm3, v1=legacy), **name**, **version**(SemVer).
선택 주요: description, type(application/library, 기본 application), appVersion(SemVer 불필요), kubeVersion(호환 K8s 범위, `>= 1.18.0 < 1.20.0`, `||` OR 가능), keywords, home, sources, **dependencies**(함께 설치), maintainers, icon, deprecated, annotations.

Guestbook 수정:
```yaml
description: An application used for keeping a running record of guests
appVersion: v5
```

- 권장: apiVersion/name/version 외 최소 appVersion·description. 공개 시 maintainers/home/sources/keywords 추가.

---

## Summary (핵심 정리)
- Helm chart = chart definition(Chart.yaml) + K8s 리소스 생성용 template 파일로 구성된다.
- YAML(key-value, map, list, 공백 기반, tab 금지)이 Helm 파일의 기본 포맷이며 JSON도 사용 가능하다.
- helm create는 NGINX 기반의 동작하는 scaffold를 생성하고, local path로 publish 없이 설치·테스트할 수 있다.
- Chart.yaml 필수 필드는 apiVersion(v2)·name·version이며, appVersion·dependencies 등 선택 필드가 chart 정체성과 Artifact Hub 표시를 결정한다.
