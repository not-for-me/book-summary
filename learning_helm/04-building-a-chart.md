# 04. Building a Chart

## 챕터 개요 (3줄 요약)
- helm create로 chart를 scaffold하고 chart 구조와 각 파일의 역할을 anvil 예제로 살펴본다.
- Chart.yaml(메타데이터), template(Deployment), values.yaml(기본값) 구성을 이해·수정한다.
- chart를 archive로 packaging하고 helm lint로 검증하는 방법을 다룬다.

---

## 1. The chart creation command
> `helm create <name>`은 best practice를 따르는 동작하는 Nginx chart를 생성한다.

```
helm create anvil
helm install myapp anvil    # 즉시 설치 가능
helm delete myapp
```
구조: Chart.yaml(메타), charts/(dependency), templates/(NOTES.txt, _helpers.tpl, deployment/service/ingress/serviceaccount.yaml, tests/), values.yaml.
- 노출 방식(K8s 리소스 타입): ClusterIP(기본, 내부 IP), NodePort(node 정적 포트), LoadBalancer(외부 LB), Ingress(HTTP/HTTPS, Ingress Controller 필요).
- 다른 시작점은 starter pack(ch6).

---

## 2. The Chart.yaml file
> chart 메타데이터를 담으며 필수 필드는 apiVersion·name·version이다.

- **apiVersion**: v2(Helm 3), v1(legacy). **name**(lowercase alnum/-/.), **version**(SemVer).
- 선택: description, type(application 기본/library), **appVersion**(앱 버전, SemVer 불필요·template에서 사용), icon(data URL 가능), keywords, home, sources, maintainers.

---

## 3. Modifying templates (Deployment)
> Helm은 Go text/template 기반으로 `{{ }}` 액션·pipeline(`|`)을 사용하며, deployment.yaml은 .Values·include로 구성된다.

```
product: {{ .Values.product | default "rocket" | quote }}   # pipeline
name: {{ include "anvil.fullname" . }}                       # _helpers.tpl 재사용
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```
- `.`=현재 scope의 root object, .Values는 그 property.
- include: 다른 template 출력을 포함(첫 인자=template명, 둘째=root object). _helpers.tpl의 anvil.fullname/labels 등.
- 값 우선순위: values.yaml < 전달 값(--set/--set-file/--set-string, -f/--values), 나중이 우선.

---

## 4. Using the values file
> values.yaml은 chart 기본값이자 설정 문서로, image·service·ingress·resource 등을 정의한다.

```yaml
replicaCount: 1
image: { repository: ..., pullPolicy: IfNotPresent, tag: "" }   # tag 빈값=appVersion
service: { type: ClusterIP, port: 80 }
ingress: { enabled: false }     # enabled로 if 토글
resources: {}                   # 권장값은 주석 처리(환경 다양성 대비)
```
- 비정형 YAML — 작성자가 구조 자유 설계. 주석으로 구조·기본값 문서화하되 비활성.
- pullPolicy: 이동 tag면 Always, 버전 고정이면 IfNotPresent. imagePullSecrets로 private registry 접근.
- serviceAccount·securityContext·nodeSelector/tolerations/affinity는 opt-in 형태로 포함.

---

## 5. Packaging & linting
> chart를 `name-version.tgz` archive로 packaging하고 helm lint로 검증한다.

```
helm package anvil [-u/--dependency-update] [-d dest] [--app-version] [--version]
helm lint anvil [mychart] [--strict]
```
- archive = gzip TAR(.tgz), `name-version.tgz` 패턴(다중 버전 공존). .helmignore로 제외 파일 지정.
- lint 피드백 3단계: info(exit 0), warning(기본 exit 0, --strict면 nonzero), error(invalid manifest, nonzero exit). archive·directory 모두 가능.

---

## Summary (핵심 정리)
- helm create는 동작하는 Nginx 기반 chart를 생성하며, Chart.yaml(apiVersion/name/version 필수)·templates/·values.yaml로 구성된다.
- template은 Go text/template 기반 `{{ }}`·pipeline·include로 K8s manifest를 생성하며 .Values로 파라미터화한다.
- values.yaml은 비정형 기본값/문서로 image·service·ingress(enabled 토글)·resource(주석 권장값)를 담는다.
- chart는 `name-version.tgz`로 packaging(.helmignore 제외)하고 helm lint(info/warning/error)로 검증한다.
