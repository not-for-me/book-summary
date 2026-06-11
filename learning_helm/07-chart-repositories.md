# 07. Chart Repositories

## 챕터 개요 (3줄 요약)
- chart repository(HTTP(S) 정적 서비스)의 internals(index.yaml)와 index 생성·병합을 다룬다.
- repository 구성(정적 서버, basic auth/mTLS 보안, GitHub Pages 실제 예시)과 helm repo 명령을 설명한다.
- 차세대 저장소인 실험적 OCI registry 지원과 관련 생태계 프로젝트를 소개한다.

---

## 1. The repository index
> 모든 chart repository는 사용 가능한 chart·버전·다운로드 위치를 나열한 index.yaml을 포함한다.

```yaml
apiVersion: v1
entries:
  superapp:
  - { name, version, appVersion, digest(SHA-256), urls: [...], created }
generated: "..."
```
- entries에 chart별 모든 버전. urls는 상대경로 또는 다른 도메인 절대 URL 가능. helm search는 최신 버전 표시.

---

## 2. Generating & merging index
> helm repo index로 디렉토리의 .tgz를 스캔해 index.yaml을 생성하고, --merge로 기존 index에 추가한다.

```
helm package superapp/ --destination charts/
helm repo index charts/                          # 새 index 생성
helm repo index workspace/ --merge index-old.yaml  # 기존에 병합(CI/CD)
```
- merge는 모든 archive 디렉토리 없이도 가능. 동시 merge 시 race condition 주의(단일 CI job 또는 ChartMuseum 동적 서버).

---

## 3. Setting up & securing a repository
> repository는 정적(Apache/Nginx/S3)으로 서빙 가능하며, basic auth·mTLS·HTTPS로 보호한다.

```
helm repo add mycharts http://localhost:8080 --username u --password p   # basic auth
helm repo add r https://... --cert-file client.crt --key-file client.key [--ca-file ...]  # mTLS
```
- 정적 서버로 index.yaml·.tgz 서빙(Python http.server 예시).
- basic auth(Authorization 헤더, HTTPS 권장), mTLS(client 인증서로 서버가 client 검증). cert 파일은 Helm 캐시에 경로 저장 → 이동 금지.

---

## 4. GitHub Pages & helm repo commands
> GitHub Pages로 무료 public repository를 호스팅하고, helm repo/pull 명령으로 사용한다.

GitHub Pages: public repo 생성 → Settings에서 Pages(main branch) 활성화 → chart package + index.yaml commit/push → `helm repo add gh-pages https://user.github.io/mycharts/`.

```
helm repo add / list [-o yaml|json] / update / remove
helm pull mycharts/superapp [--version 0.1.0]
helm install superapp-dev mycharts/superapp
```

---

## 5. Experimental OCI support
> Helm 3는 OCI Distribution Spec 기반 container registry에 chart를 저장하는 실험적 지원을 추가했다(namespace·세밀한 access control 등 기존 한계 해결).

```
export HELM_EXPERIMENTAL_OCI=1
helm registry login -u myuser localhost:5000
helm chart save mychart/ localhost:5000/myrepo/mychart   # 캐시 저장
helm chart push / pull / list / export / remove
```
- tag는 Chart.yaml version 기반(또는 `:stable` 커스텀). 관련 프로젝트: ChartMuseum(동적 repo 서버, 다양한 storage backend), Harbor(security registry), Chart Releaser(cr, GitHub releases), S3/GCS/Git plugin.

---

## Summary (핵심 정리)
- chart repository는 index.yaml(chart·버전·digest·urls)을 가진 HTTP(S) 정적 서비스다.
- helm repo index로 index를 생성하고 --merge로 CI/CD 환경에서 병합한다(race condition 주의).
- 정적 서버로 서빙하며 basic auth·mTLS·HTTPS로 보호하고, GitHub Pages로 무료 public 호스팅이 가능하다.
- Helm 3는 OCI registry에 chart를 저장하는 실험적 기능(save/push/pull)을 제공하며 ChartMuseum·Harbor 등 생태계가 있다.
