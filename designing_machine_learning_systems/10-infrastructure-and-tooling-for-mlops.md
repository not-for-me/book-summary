# 10. Infrastructure and Tooling for MLOps

## 챕터 개요 (3줄 요약)
- ML infrastructure는 storage·compute, resource management, ML platform, dev environment 4계층으로 구성되며 필요 수준은 규모·use case에 따라 다르다.
- compute는 cloud(elastic, 초기 유리)와 private data center(cloud repatriation, 성장 시 비용 절감) 사이 선택이며, dev environment는 표준화·container화가 중요하다.
- resource management(scheduler/orchestrator), ML platform(deployment/model store/feature store), build vs buy 결정을 다룬다.

---

## 1. Infrastructure 개요
> ML system은 복잡할수록 좋은 infrastructure로 이득. 규모 스펙트럼: ad hoc 분석(Jupyter면 충분) ~ 특수 요구(self-driving, Google Search; 자체 구축) ~ 중간(reasonable scale, GB~TB/일; 표준화된 ML infra 이득).
> 4계층: storage·compute, resource management(Airflow/Kubeflow/Metaflow), ML platform(SageMaker/MLflow), dev environment.

## 2. Storage and Compute
> storage는 commoditized·cloud화(저렴해 다 저장). **compute layer**: 사용 가능 compute 자원과 사용 메커니즘. 작게 쪼개 동시 사용(CPU thread, Spark "job", K8s "pod").
> compute unit 특성: memory(GB)와 operation 속도(FLOPS — 논쟁적, utilization = 실제 FLOPS/가능 FLOPS, I/O bandwidth 중요). MLPerf로 benchmark. 실무엔 core 수+memory로 평가(AWS vCPU ≈ 물리 core 절반).

## 3. Public Cloud vs Private Data Centers
> cloud: 사용량만큼 지불, bursty한 ML workload(실험 시 surge)에 유리, autoscaling. 단 무한 아님(instance 한도), 성장 시 비용 급증(public software 기업 cost of revenue ~50%).
> **cloud repatriation**: 자체 data center로 이동(Dropbox IPO 전 2년간 $75M 절감). 대부분 hybrid. **multicloud**: vendor lock-in 회피(81% 조직이 2+ provider)이나 의도보다 우연(조직 독립·인수·전략 투자)에 기인.

## 4. Development Environment
> ML engineer가 code 작성·실험·production 상호작용하는 곳. 구성: IDE, versioning(Git/DVC/W&B/MLflow), CI/CD. 가장 과소투자되나 생산성에 직결("infra 하나만 잘 set up한다면 dev environment").
> **IDE**: VS Code/Vim(native), Cloud9(browser). **notebook**(Jupyter/Colab): 임의 artifact 포함, stateful(실패 step부터 재실행)이나 out-of-order 실행으로 재현성 어려움. Papermill, Commuter, nbdev.

## 5. Standardizing Dev Environments & Containers
> dev environment를 팀/회사 단위 표준화(package 버전·Python 버전·machine 통일). cloud dev environment(local IDE + SSH 또는 cloud IDE)로 이동 → IT 지원·remote·보안·dev-prod 격차 축소 이점.
> **container**(Docker): Dockerfile(환경 재현 instruction) → image → container. 단계별 의존성 충돌·자원 차이 시 별도 container. **container orchestration**: Docker Compose(단일 host), Kubernetes(다중 host, network·자원 공유·고가용성).

```
Dockerfile (recipe) -> Docker image (mold) -> Docker container (instance)
```

## 6. Resource Management
> pre-cloud: 유한 자원 활용 극대화. cloud: 비용 효율화로 초점 이동(자원 추가가 타 app 감소 아님). engineer 시간 > compute 시간이면 자동화로 생산성 우선.
> ML workflow 특성: **repetitiveness**(주기적 train/predict → cron)와 **dependencies**(복잡 의존 → DAG).
> **cron**: 정해진 시간 실행(의존성 무시). **scheduler**(Slurm, Borg): DAG·의존성·event trigger·실패 처리·queue·자원 인식. **orchestrator**(K8s): 어디서 자원 확보(machine/instance/cluster provisioning). scheduler는 주기 job, orchestrator는 long-running service. 종종 scheduler가 orchestrator 위에서 실행.

## 7. Data Science Workflow Management
> workflow를 DAG로 정의(code Python 또는 config YAML), 각 step = task. scheduler + orchestrator로 자원 할당.
> **Airflow**: 최초기, "configuration as code"(Python), 풍부한 operator. 단점: monolithic, DAG 비parameterized·static.
> **Prefect**: parameterized·dynamic(Python). container가 우선순위 아님.
> **Argo**: 각 step 자체 container(YAML 정의). 단 K8s에서만(local은 minikube).
> **Kubeflow/Metaflow**: dev·prod 모두 실행, infra boilerplate 추상화. Kubeflow Pipelines(Argo 위). Metaflow는 @conda/@batch decorator로 step별 환경·자원 지정, dev-prod 동일 code(UX 우수).

## 8. ML Platform — Deployment & Model Store
> ML 응용 전반에 공유되는 tool 집합(2020년 이후 트렌드). 평가 기준: cloud provider 호환, open source vs managed service.
> **model deployment**: model+의존성을 production에 push하고 endpoint 노출. 가장 성숙(SageMaker, Vertex AI, Seldon 등). online은 쉬우나 batch는 까다로움(종종 별도 pipeline). test in production 지원 확인.
> **model store**: blob 저장만으론 부족(production-local 불일치 debugging 곤란). 8 artifact 추적: model definition, parameter, featurize/predict 함수, dependency, data, model generation code, experiment artifact, tag. MLflow가 대표적이나 artifact 문제 미해결 → Stitch Fix 등 자체 구축.

## 9. Feature Store & Build vs Buy
> **feature store** 3문제: ① feature management(공유·발견·권한; Uber 10,000 feature; Amundsen/DataHub), ② feature computation(계산·저장, data warehouse 역할), ③ feature consistency(batch/streaming feature 통일로 train-inference 불일치 방지). Feast(batch 강점), Tecton, SageMaker/Databricks. 조사 기업 40%만 사용, 그 중 절반은 자체 구축.
> **build vs buy** 요인: ① 회사 단계(초기엔 vendor로 빠르게, 성장 시 자체가 저렴), ② 경쟁우위 초점("잘하고 싶은 것은 in-house"), ③ tool 성숙도. build가 항상 싸진 않음(engineer 채용·유지, 혁신 비용·integration hell).

---

## Summary (핵심 정리)
- ML을 production에 올리는 것은 infrastructure 문제이며, storage·compute, dev environment, resource management, ML platform 4계층을 규모에 맞게 set up해야 한다.
- storage·compute는 commoditized되어 cloud가 초기 유리하나 성장 시 cloud repatriation을 고려하고, dev environment는 표준화·container(Docker/K8s)화로 dev-prod 격차를 줄인다.
- resource management는 cron→scheduler→orchestrator로 진화하며 ML workflow는 DAG 기반 workflow 관리 tool(Airflow/Argo/Metaflow)을 쓰고, ML platform(deployment/model store/feature store)과 build vs buy 결정이 핵심이다.
