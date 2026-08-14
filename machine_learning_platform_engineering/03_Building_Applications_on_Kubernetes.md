# 03. Building Applications on Kubernetes

## 챕터 개요 (3줄 요약)

- ML platform의 infrastructure backbone인 Docker(containerization), Kubernetes(orchestration), CI/CD, monitoring을 hands-on으로 다룬다.
- FastAPI joke app 하나를 containerize → K8s deploy → GitLab CI/Argo CD로 자동화 → Prometheus/Grafana로 monitoring하는 전 과정을 실습한다.
- 이 tool들은 ML 전용은 아니지만 scale에서 robust ML system을 build하는 foundation이며, 이후 챕터의 ML-specific tooling이 이 위에 얹힌다.

---

## 3.1 Containers and tooling

> model을 일관·확장 가능하게 배포하려면 containerization과 orchestration tooling이 필요하다.

- container는 on-premise/cloud에 배포되며 충분한 resource·auto-scaling·failure 시 교체가 자동화·모니터링되어야 함.
- tool stack: **Docker**(containerization) → **Kubernetes**(resource 관리·scaling) → **GitLab CI + Argo CD**(CI/CD) → **Prometheus + Grafana**(monitoring·visualization).
- Kubernetes 선택 이유: scalable·portable·cloud-agnostic orchestration layer. tool은 수단일 뿐, 목표(자동화된 ML platform) 달성이 핵심.

---

## 3.2 Docker

> Docker는 app+dependency를 하나의 container로 packaging해 어느 환경에서도 동일하게 실행되게 한다.

### 개념 (shipping container 비유)

- application = product, dependency/config = handling instruction, Docker container = 표준 shipping container. 표준화로 어느 환경에서도 seamless 배포.

### Dockerfile / image / registry

- **Dockerfile**: build instruction 목록. `FROM`(base image) → `WORKDIR` → `RUN`(패키지 설치) → `COPY` → `ARG`/`ENV` → `ENTRYPOINT`. 각 instruction = layer.
- **image**: 배포 준비된 package. `docker build . -t hello-joker:v1`. tag는 version 또는 Git commit ID.
- **container**: image의 runnable instance. `docker run -it -p 8081:8083 hello-joker:v1`.
- **container registry**: image 저장소 (Docker Hub, GCR, ECR). K8s가 여기서 pull. `docker push <user>/hello-joker:v1`.

```dockerfile
FROM python:3.10-slim-buster
WORKDIR /app
RUN apt-get update && apt-get install -y build-essential
COPY requirements.txt /app/requirements.txt
RUN pip3 install --no-cache-dir -r requirements.txt
COPY . /app
EXPOSE 8083
ENTRYPOINT ["/app/entrypoint.sh"]
```

---

## 3.3 Kubernetes (K8s)

> K8s는 container orchestrator로, scheduling·scaling·failure 복구를 자동화한다.

### 3.3.1 Architecture

- **cluster** = master + worker node들 (resilience·scaling).
- **Master node** service: **API server**(frontend, 요청 수신), **etcd**(분산 key-value store, single source of truth), **Scheduler**(resource 기반 node 배정), **Controller**(orchestration, 죽은 container 재기동).
- **Worker node** service: **Container runtime**(Docker), **Kubelet**(node/container 상태를 master에 보고·명령 수행).

```text
User -> kubectl -> [API server] -- etcd / Scheduler / Controller (master)
                        |
                        v
              [Kubelet + Container runtime] (worker nodes)
```

### 3.3.2 kubectl

- cluster 상호작용 CLI: `kubectl get nodes`, `cluster-info`, `run`, `create`, `apply`, `describe`, `logs`, `port-forward`, `delete`.

### 3.3.3 K8s objects (YAML: apiVersion / kind / metadata / spec)

- **Pod**: 최소 단위, app 1 instance. multi-container pod은 utility container(logging 등)가 main과 생사 공유.
- **ReplicaSet**: pod 모니터링·replica 유지. selector(label match)로 기존 pod도 관리. `replicas`, `template`, `selector`.
- **Deployment**: ReplicaSet의 상위 추상화. **rolling update**(한 pod씩 교체, error 시 pause), **rollback**(`kubectl rollout undo`). stateless app·ML inference service에 주로 사용.

```text
Deployment --> ReplicaSet --> Pods (application container)
   (rolling update + rollback)   (desired replica count)
```

### 3.3.4 Networking and services

> pod는 IP를 갖지만 transient → service가 stable abstraction layer 제공.

- **ClusterIP**: cluster 내부 전용 IP (frontend↔backend 내부 통신). DNS name = service name.
- **NodePort**: node IP + port(30000~32767)로 외부 접근. dev/test용.
- **LoadBalancer**: cloud provider의 external load balancer + external IP. public-facing production용.
- port 개념: `targetPort`(pod), `port`(service), `nodePort`(외부 접근).

### 3.3.5 Other objects

- **Namespace**: 물리 cluster 내 virtual cluster, resource isolation·naming 충돌 방지. cross-namespace 호출: `<svc>.<ns>.svc.cluster.local`.
- **ConfigMap**: config data(DB URL, API endpoint)를 code와 분리, key-value. `configMapKeyRef`로 env 주입.
- **Secret**: 민감 정보. base64 **encoding**(암호화 아님!) — 진짜 암호화는 Vault 등 외부 도구 권장. type: Opaque/Docker Registry/TLS. `secretKeyRef`로 주입.

### 3.3.6 Helm charts

- K8s **package manager**. 여러 object(deployment+service+configmap+secret)를 하나의 **chart**(template + values.yaml)로 packaging.
- **Artifact Hub** = chart repository (Docker Hub의 chart 버전). 명령: `helm repo add/update`, `helm search`, `helm install --generate-name`, `helm list`, `helm uninstall`, `helm pull --untar`, `helm create`.
- values.yaml로 replicaCount·image·service type 등 customize.

### 3.3.7 K8s와 ML

- ML workload는 compute-intensive → K8s의 효율적 scaling·resource 관리가 특히 적합.

---

## 3.4 Continuous integration and deployment

> 수동 배포는 잘못된 image tag·overwrite 위험 → CI/CD로 자동화하고 Git을 single source of truth로.

### 3.4.1 GitLab CI

- commit마다 trigger. 3 stage: **test**(pytest) → **build**(docker build/push, tag = 짧은 commit SHA) → **update**(yq로 Helm values.yaml image tag 갱신 후 push).
- predefined var: `CI_PROJECT_DIR`, `CI_COMMIT_SHA`. 민감정보는 Settings > CI/CD > Variables + access token(scope: API).
- `[skip ci]` commit message로 무한 CI loop 방지. GitLab Runner로 local 테스트.

```yaml
stages: [test, build, update]
test_code:   { stage: test,   script: [pytest $CI_PROJECT_DIR/] }
build_image: { stage: build,  script: [docker build ..., docker push ...] }
update_helm_chart: { stage: update, script: [yq ... values.yaml, git push] }
```

### 3.4.2 Argo CD

- CD tool이자 Git↔production sync 보장. Git이 **only source of truth**.
- production에서 수동 변경 시 Git 기준으로 rollback (auto-sync 시 자동). Helm chart 경로를 source로 지정, New App으로 배포.
- Git과 K8s manifest 차이 발생 시 "out of sync" 표시, diff로 확인.

```text
Git repo (source of truth) <== monitor/sync ==> Argo CD ==> K8s cluster
```

---

## 3.5 Prometheus and Grafana

> logging만으로 답할 수 없는 "지난 1시간 요청 수" 같은 질문은 metric 수집이 필요하다.

- **Prometheus**: pull-based monitoring. 4 component — time series DB, scraping worker, UI, **Alertmanager**(email/push 등 alert 라우팅). app이 `/metrics` endpoint 노출 → worker가 scrape.
- **Grafana**: dashboarding tool. Prometheus를 data source로 **PromQL** query로 시각화.
- metric type: **counter**(증가만, 요청 수), **gauge**(증감, memory), **histogram**(bucket, 95th percentile 응답시간).
- FastAPI는 `starlette_exporter`(PrometheusMiddleware + /metrics route)로 손쉽게 노출. custom metric은 `Counter(...).inc()`.
- Prometheus **service discovery**: pod annotation `prometheus.io/scrape/path/port`로 자동 발견.

```python
from starlette_exporter import PrometheusMiddleware, handle_metrics
app.add_middleware(PrometheusMiddleware)
app.add_route("/metrics", handle_metrics)
```

```text
App(/metrics) -> Prometheus(scrape + TSDB) -> Grafana(PromQL viz)
                          |
                          +-> Alertmanager -> email/push
```

---

## Summary (핵심 정리)

- ML platform 작업에는 automation·deployment·monitoring을 다루는 DevOps tooling의 실무 지식이 중요하다.
- Docker로 app을 containerize하고, Kubernetes로 production container를 orchestrate한다.
- CI/CD(GitLab CI + Argo CD)로 build·deploy를 자동화하면 manual error 감소, 빠른 cycle, Git 기반 single source of truth 확보.
- Prometheus/Grafana monitoring으로 production app의 reliability·performance 유지.
- tool은 수단일 뿐 — 조직 목표에 맞게 선택·통합·활용하는 것이 관건. 다음 챕터(4장)부터 이 foundation 위에 ML-specific tooling(feature store, pipeline, drift monitoring)을 쌓는다.