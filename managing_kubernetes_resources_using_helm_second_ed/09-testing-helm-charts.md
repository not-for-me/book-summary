# 09. Testing Helm Charts

## 챕터 개요 (3줄 요약)
- Helm chart의 template 렌더링·linting을 local에서 검증하는 방법(helm template/lint, yamllint)을 다룬다.
- live cluster에서 readiness probe(--wait)와 test hook(helm test)으로 동작을 검증한다.
- monorepo 환경에서 lint·install·upgrade·versioning을 자동화하는 Chart Testing(ct) 도구를 소개한다.

---

## 1. Verifying Helm templating
> helm template으로 리소스 생성 결과를 local 렌더링하고, --validate/--dry-run으로 server-side 검증을 추가한다.

```
helm template my-guestbook guestbook            # client-side 렌더링
helm template my-release <chart> --validate     # API server 검증
helm install my-chart <chart> --dry-run         # 설치 전 sanity check
```
- 검증 대상: 파라미터 치환, if/range/with, indentation, function/pipeline, required/fail/schema, dependency.

---

## 2. Linting (helm lint & yamllint)
> helm lint는 Chart.yaml 유효성·구조를, yamllint는 YAML style을 검사한다.

```
helm lint <chart>                               # Chart.yaml/구조, INFO/WARNING/ERROR
helm template my-guestbook <chart> | yamllint -  # YAML style(들여쓰기·길이·trailing space)
```
- helm lint: Chart.yaml 필수 필드(name/apiVersion/version), values.yaml·templates 존재, 파일 확장자 확인. 렌더 리소스나 YAML style은 미검증.
- yamllint: `.yamllint.yaml`로 규칙 커스터마이즈.

---

## 3. Testing in a live cluster
> readiness probe + `--wait`로 설치 성공을 판정하고, `helm.sh/hook: test` pod + `helm test`로 기능을 smoke test한다.

```
helm install guestbook <chart> -n chapter9 --wait   # probe 통과까지 block(기본 5분 timeout)
helm test guestbook -n chapter9 --logs              # test hook 실행 + 로그
```
- readiness probe: 성공 시 pod Ready. --wait는 probe 실패 시 exit 1.
- test hook: templates/tests/에 정의, helm test 시 생성·실행(예 frontend wget 호출).

---

## 4. Chart Testing (ct) tool
> ct는 Git monorepo에서 변경된 chart만 자동 lint·validate·install·upgrade하고 version 증가를 강제한다.

ct 명령: lint / install / **lint-and-install** / list-changed.

```
ct lint-and-install            # 변경 chart lint+install+test hook, version 증가 확인
ct lint-and-install --upgrade  # MAJOR 동일 시 이전→신버전 in-place upgrade 회귀 테스트
```

- 변경 감지: Git diff. 변경 chart는 Chart.yaml version 증가 필수(SemVer: MAJOR=breaking, MINOR=feature, PATCH=fix).
- **ci/** 폴더의 `*values.yaml`마다 다른 값 조합으로 반복 lint·install.
- 설정: ct.yaml(chart-dirs, chart-repos), lintconf.yaml(yamllint), chart_schema.yaml(yamale schema).
- 의존 도구: helm, git, kubectl, yamllint, yamale.
- --upgrade 시 MAJOR가 바뀌면 upgrade 테스트 skip.

---

## Summary (핵심 정리)
- 기본 검증은 helm template(렌더, --validate/--dry-run)와 helm lint(Chart.yaml/구조) + yamllint(YAML style)다.
- live 검증은 readiness probe(--wait)와 test hook(helm test)으로 수행한다.
- ct는 monorepo에서 변경된 chart만 자동 테스트하고 SemVer version 증가를 강제하며, ci/ 폴더로 다중 값 조합을 검증한다.
- ct --upgrade는 MAJOR가 같을 때 in-place upgrade로 회귀(backward compatibility)를 확인한다.
