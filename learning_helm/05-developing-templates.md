# 05. Developing Templates

## 챕터 개요 (3줄 요약)
- Go text/template 기반 Helm template 문법(action, pipeline, 전달 데이터 객체)을 다룬다.
- function·method, if/else/with·variable·loop(range)·named template으로 template을 프로그래밍한다.
- template 유지보수 구조와 debugging(dry run, get manifest, lint) 기법을 설명한다.

---

## 1. Template syntax & passed-in data
> `{{ }}` action 안에서 로직을 작성하며(`-`로 공백 제거), Helm은 단일 데이터 객체 `.`(dot)를 전달한다.

전달 데이터:
- **.Values**: values.yaml + 전달 값(스키마 없음, chart별 상이).
- **.Release**: Name/Namespace/IsInstall/IsUpgrade/Service.
- **.Chart**: Chart.yaml 필드(Name/Version/AppVersion/Annotations, 대문자 시작). 커스텀 정보는 annotations로.
- **.Capabilities**: APIVersions.Has, KubeVersion(helm template 시 기본값).
- **.Files**: 비특수·비-helmignore 파일. **.Template**: Name/BasePath.
- scope 변경 시에도 `$`로 root(`$.Capabilities...`) 접근.

---

## 2. Pipelines, functions, methods
> pipeline(`|`)으로 함수를 연결하며, Helm/Sprig 함수와 .Capabilities/.Files method를 제공한다.

```
character: {{ .Values.character | default "Sylvester" | quote }}
{{- toYaml .Values.podSecurityContext | nindent 8 }}   # 데이터→YAML, 들여쓰기
id: "12345e2"   # quote로 scientific notation 버그 방지
```
- 100+ 함수(Sprig): math, dict/list, hash, date, toYaml/toJson/toToml, indent/nindent. 마지막 인자로 pipeline 입력.
- method: `.Capabilities.APIVersions.Has "apps/v1/Deployment"`, `.Files.Get/GetBytes/Glob/AsConfig/AsSecrets/Lines`.
- **lookup** "apps/v1" "Deployment" "ns" "name": cluster 리소스 조회(dry run·template에선 빈 결과).

---

## 3. Control structures & variables
> if/else, with(scope 변경), range(loop), `$var := / =` variable로 흐름과 데이터를 제어한다.

```
{{- if and .Values.a .Values.b -}} ... {{- else -}} ... {{- end }}  # and/or=함수, 괄호로 그룹
{{- with .Values.ingress.annotations }}{{- toYaml . | nindent 4 }}{{- end }}  # scope=.
{{- range $key, $value := .Values.products }}- {{ $key }}: {{ $value }}{{- end }}
{{ $var := .Values.character }}  # 생성, $var = "..." 변경
```
- if: pipeline 결과 truthy면 실행, end 필수. with: 값 있으면 scope를 그 값으로 변경(빈 값이면 skip).
- range: list/dict 반복. `.`만 변경하거나 `$key,$value` 변수 생성.

---

## 4. Named templates & debugging
> define으로 재사용 template을 만들고 include로 호출하며, dry run·get manifest·lint로 디버깅한다.

```
{{- define "anvil.selectorLabels" -}} ... {{- end -}}
{{- include "anvil.selectorLabels" . | nindent 6 }}   # pipeline 가능(template은 불가)
```
- 이름은 chart명으로 namespace화(collision 방지). _helpers.tpl에 모음(`_`로 상단 정렬). 복잡 로직(예 getImage: digest>tag>appVersion) 캡슐화.
- 구조 권장: manifest별 별도 파일(deployment.yaml, statefulset-primary/replica.yaml), named template은 _helpers.tpl.
- debugging: **--dry-run**(렌더+K8s schema 검증, HOOKS/MANIFEST 출력), **helm template**(YAML만, schema 미검증), **helm get manifest**(설치된 원본), **helm lint**(이름 규칙 등 API 외 문제).

---

## Summary (핵심 정리)
- Helm template은 Go text/template 기반 `{{ }}` action·pipeline으로, .Values/.Release/.Chart/.Capabilities/.Files/.Template 데이터를 받는다.
- function(Helm/Sprig)·method(.Files, .Capabilities, lookup)로 데이터를 변환·조회하고, toYaml|nindent 패턴을 자주 쓴다.
- if/else/with(scope)·range(loop)·variable로 흐름을 제어하고, define/include로 재사용 template(_helpers.tpl)을 만든다.
- dry-run(schema 검증)·helm template·get manifest·lint로 template을 디버깅한다.
