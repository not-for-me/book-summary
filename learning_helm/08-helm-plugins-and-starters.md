# 08. Helm Plugins and Starters

## 챕터 개요 (3줄 요약)
- Helm을 확장하는 두 방법인 plugin(custom subcommand)과 starter(chart 템플릿)를 다룬다.
- plugin 설치·빌드(plugin.yaml, hook, downloader, 실행 환경 변수, shell completion)를 설명한다.
- starter로 helm create 시 커스텀 chart 출발점을 만드는 방법을 다룬다.

---

## 1. Plugins overview & installing
> plugin은 Go 소스 수정 없이 Helm CLI에 custom subcommand를 추가하는 외부 도구다.

```
helm plugin install https://github.com/salesforce/helm-starter.git [--version v3.1.0]
helm plugin list / update / remove
```
- VCS URL(git/svn/bzr/hg), tarball URL, local 디렉토리(symlink, 개발용)에서 설치.
- 예시 plugin: helm-2to3, helm-secrets, helm-backup, helm-schema-gen, helm-mapkubeapis.
- 설치 후 plugin 이름이 top-level subcommand가 됨(`helm <plugin> [args]`, helm help에도 표시).

---

## 2. Building a plugin (plugin.yaml)
> plugin은 plugin.yaml(메타+invocation command)과 구현 스크립트로 정의된다(언어 무관).

```yaml
name: inspect-templates
version: 0.1.0
description: ...
command: "${HELM_PLUGIN_DIR}/inspect-templates.sh"
platformCommand: [{ os: windows, arch: amd64, command: "bin/x.exe" }]
ignoreFlags: false
hooks: { install: ..., update: ..., delete: ... }
downloaders: [{ command: ..., protocols: [myp, myps] }]
```
- command 결정 순서: platformCommand(os+arch > os) → 기본 command.
- name은 기존 subcommand와 충돌 금지(a-zA-Z0-9_-). version은 SemVer2.
- `$(helm env HELM_PLUGINS)` 디렉토리에 plugin명 폴더 + plugin.yaml + 실행 스크립트 수동 설치 가능.

---

## 3. Hooks, downloaders, execution env
> hook으로 install/update/delete 시 작업하고, downloader plugin으로 custom protocol 다운로드를 구현하며, 런타임 환경 변수를 받는다.

- **hooks**: install/update/delete 시 스크립트 실행(예 OS별 binary 다운로드).
- **downloader**: custom protocol(`ss://`) 선언 → Helm이 해당 plugin으로 index.yaml/.tgz 다운로드. 인자 `<cmd> certFile keyFile caFile full-URL`, 결과는 stdout(로그는 stderr). 예: bearer token auth repo.
- 환경 변수: HELM_BIN, HELM_DEBUG, HELM_NAMESPACE, HELM_PLUGIN_DIR, HELM_PLUGINS, HELM_REPOSITORY_CACHE/CONFIG 등.

---

## 4. Shell completion & starters
> plugin은 static(completion.yaml)/dynamic(plugin.complete) shell completion을 제공하고, starter는 새 chart의 템플릿이 된다.

- **static completion**: completion.yaml에 flags/commands/validArgs 정의.
- **dynamic completion**: `plugin.complete` 실행파일이 Tab 시 가능한 결과를 줄단위 출력(예 cluster release 조회). static 있으면 dynamic 미사용.
- **starter**: `helm create --starter <name> <chart>`. chart를 starter로 변환 시 하드코딩된 이름을 `<CHARTNAME>`으로 치환, `$(helm env HELM_DATA_HOME)/starters`에 배치.

---

## Summary (핵심 정리)
- plugin은 Go 수정 없이 Helm CLI에 custom subcommand를 더하는 외부 도구로, VCS/tarball/local에서 설치한다.
- plugin.yaml이 메타·invocation command·platformCommand·hook·downloader를 정의하며 구현은 언어 무관이다.
- hook(install/update/delete), downloader plugin(custom protocol), 런타임 환경 변수, static/dynamic shell completion으로 통합을 확장한다.
- starter는 helm create의 커스텀 출발점으로, 이름을 `<CHARTNAME>`으로 치환해 HELM_DATA_HOME/starters에 둔다.
