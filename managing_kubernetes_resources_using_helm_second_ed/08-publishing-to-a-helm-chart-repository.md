# 08. Publishing to a Helm Chart Repository

## 챕터 개요 (3줄 요약)
- Helm chart repository의 두 구현(HTTP server, OCI registry)과 구성 요소를 설명한다.
- guestbook chart를 GitHub Pages(HTTP)에 package/index/upload 절차로 publish한다.
- guestbook chart를 OCI registry(GitHub Container Registry)에 helm push/pull로 publish·다운로드한다.

---

## 1. Understanding Helm chart repositories
> chart repository는 chart를 community에 배포하는 곳으로, HTTP server 또는 OCI registry로 구현한다(공개 검색은 Artifact Hub).

- **HTTP server**: 가장 오래된 일반적 방식. Apache httpd, NGINX, S3, GitHub Pages, ChartMuseum(API·자동 index) 등.
  - 구성: chart .tgz archive + **index.yaml**(chart metadata).
- **OCI registry**: OCI Artifacts로 container image와 같은 registry에 chart 저장 → 도구 공유, 노력 절감.

---

## 2. Publishing to an HTTP repository (GitHub Pages)
> chart를 .tgz로 package → index.yaml 생성 → server에 upload하는 3단계로 publish한다.

```
helm package chapter8/guestbook --dependency-update   # dep 포함 .tgz 생성
cp guestbook-0.1.0.tgz <pages-clone>
helm repo index <pages-clone>     # index.yaml 생성
git add --all && git commit -m "..." && git push origin main
helm repo add example <Pages Site URL>
helm search repo guestbook        # 검증
```

- GitHub Pages: Settings → Pages → Source=main으로 static site 활성화(Public 필요).
- index.yaml: entries에 chart 이름·version·dependencies·digest·urls 등 metadata 수록.

---

## 3. Publishing to an OCI registry
> OCI registry publish는 container image workflow와 유사하며(login/push/pull), `oci://` 프로토콜과 chart 이름·version 기반 자동 naming을 사용한다.

```
helm registry login ghcr.io                       # username + PAT
helm push guestbook-0.1.0.tgz oci://ghcr.io/<OWNER>
helm pull oci://ghcr.io/<OWNER>/guestbook --version 0.1.0
```

- 명령 대응: registry login/logout, push, pull. show/template/install/upgrade도 oci:// 사용 가능.
- v3.8.0+ 정식 지원(이전은 HELM_EXPERIMENTAL_OCI=1).
- naming: `ghcr.io/<OWNER>/<chart>:<version>` — repository·tag가 Chart.yaml의 name·SemVer로 자동 결정.
- GitHub: PAT(read/write/delete:packages) 필요. 신규 package는 private 기본 → 필요 시 visibility 변경.
- 서명(.prov) 파일이 같은 디렉토리에 있으면 signed chart push 지원.

---

## Summary (핵심 정리)
- chart repository는 HTTP server(.tgz + index.yaml) 또는 OCI registry(container image와 공존) 두 방식으로 만든다.
- HTTP publish: helm package → helm repo index → server upload(GitHub Pages 예시), repo add로 검증.
- OCI publish: helm registry login → helm push(oci://), helm pull로 다운로드. naming은 Chart.yaml의 name·version 기준 자동.
- OCI 정식 지원은 v3.8.0+이며, oci:// 프로토콜로 install/upgrade 등도 상호 호환된다.
