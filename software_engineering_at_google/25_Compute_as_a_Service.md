# 25. Compute as a Service

## 챕터 개요 (3줄 요약)
- Compute as a Service(CaaS)는 "내 코드를 실행할 하드웨어를 달라"는 단순 개념을, 조직이 성장·진화해도 살아남고 확장되는 시스템으로 매핑하는 것이다.
- 핵심은 toil 자동화 → 컨테이너화·멀티테넌시 → rightsizing/autoscaling으로 진화한 관리형 컴퓨트(Borg)이며, "cattle, not pets" 모델이 Google 성공의 토대다.
- AI 시대에 워크로드를 대규모로 운영할수록, 공통 인프라 위에서 소프트웨어를 분산·관리 환경에 맞게 작성하고 적절한 추상화 수준의 컴퓨트 서비스를 선택하는 것이 시니어의 핵심 역량이다.

---

## 1. Taming the Compute Environment (컴퓨트 환경 길들이기)
> Borg는 오늘날 CaaS(Kubernetes·Mesos)의 선구자 — naive SSH/SFTP 모델이 규모에서 붕괴하는 과정을 추적.

- toil 자동화: 배포 스크립트화 → 모니터링 자동화 → autohealing(헬스체크 실패 시 kill·재생성).
- 자동 스케줄링: 중앙 서비스가 가용 머신을 알고 빈 머신에 자동 배포(수동 "sign-up 파일" 제거); 머신 장애 감지·자동 마이그레이션.
- Containerization & Multitenancy: 1:1 머신-프로그램 매핑은 비효율 → 자원 요구사항 명시 후 bin-packing.
  - 격리(isolation) 필요: "이웃의 개가 내 RAM에서 짖는다" — CPU/RAM/의존성/보안 충돌; VM은 오버헤드 커서 Borg는 container(cgroups+chroot) 선택.
- Rightsizing & Autoscaling: 사람이 자원 수치를 정하는 건 결함 — 시간이 지나며 비효율; 자동화(Google은 Borg 사용량 절반+를 rightsizing이 결정).

---

## 2. Writing Software for Managed Compute (관리형 컴퓨트용 소프트웨어)
> 손으로 관리하는 머신 목록 → 자동 스케줄링으로의 전환은 소프트웨어 작성 방식을 근본적으로 바꿨다.

- Architecting for Failure: 스케줄러가 워커를 임의로 kill·이동 가능 → "pets vs cattle".
```
Pets vs Cattle
Pets: named, nursed back to health by a human when broken (linear+ maintenance)
Cattle: replica001..N, automation kills & re-provisions a new one (self-healing)
- divide work into small chunks, assign dynamically (lose at most 1 chunk)
- serving jobs: graceful drain on reschedule (load balancer redirects)
```
- Batch vs Serving: batch(throughput, 단명)는 MapReduce/Flume로 동적 청크 할당; serving(latency, 장수)은 자연히 load balancing에 적합하나 leader/sharded/hostname-bound 예외.
- Managing State: 인프로세스 상태는 transient — 영속 상태는 외부 복제 스토리지(GFS/Bigtable/Spanner)로; 캐시는 transient(latency용으로 provision하되 핵심은 total load로).
- Connecting to a Service: hostname 하드코딩 금지 → service discovery(간접 계층)+load balancing; 멱등성(idempotency, client-assigned ID)으로 재시도 안전화.
- One-Off Code: 일회성 분석에 분산 컴퓨트 — 엔지니어 시간이 컴퓨트보다 비싸므로(quota로 사고 방지, 저우선 batch는 사실상 무료).

---

## 3. CaaS Over Time and Scale (시간과 규모)
> 컨테이너는 격리 메커니즘일 뿐 아니라 컴퓨트 환경의 추상화 경계를 제공한다.

- Container as Abstraction: 머신이 변해도 container 소프트웨어(단일 팀 관리)만 적응; 파일시스템 추상화(외부 SW 온보딩·의존성 prepackage), named resource(네트워크 포트 — Docker/K8s가 namespace로 충돌 해결).
- 암묵적 의존성(Hyrum's Law): PID 공간 고갈 사례 — PID가 5자리(0~32,000)라는 사실에 로그 시스템이 의존 → PID namespace 도입을 8년째 진행 중; 검증된 커뮤니티 컨테이너 시스템 사용의 가치.
- One Service to Rule Them All: Borg가 batch+serving을 단일 풀로 통합 → 관리 관행 분기 방지(선형 비용 회피); serving의 overprovision slack(70%)에 batch를 채워 사실상 batch를 무료로 실행.
  - serving 멀티테넌시 요구: rescheduling throttle, kill 전 경고(grace period).
- Submitted Configuration: 문서·tribal knowledge보다 전용 config 언어로 — 다중 DC·staging/prod·부속 서비스(memcached)를 표준 config로 관리(배포 자동화의 전제).

---

## 4. Choosing a Compute Service (컴퓨트 서비스 선택)
> 대부분 조직은 직접 만들지 않고 선택하지만, 컴퓨트 인프라는 lock-in이 강하다(Hyrum's Law·헬퍼 도구 생태계).

- Centralization vs Customization: 단일 CaaS가 관리·자원 효율 최적이나(Borg), 성장하면 다양한 니즈 발생 — GCE의 VM live migration, Search의 디스크 장애 대응 등 bifurcation이 Borg API를 비대화 → 2012년 이후 API 정리(whitelist 등).
- Level of Abstraction (Serverless): 추상화↑이 항상 좋은 건 아님.
```
Abstraction ladder
bare-metal pets -> VMs (IaaS) -> containers as cattle (Borg/K8s) -> serverless
- serverless: framework multitenant, load/unload action code, "no servers"
- requires TRULY stateless code; scales down to zero (cost scales with traffic)
- natural comparison: persistent containers (Borg/K8s), not "VMs as pets"
```
  - Serverless 장단점: 저트래픽서 0까지 스케일·낮은 관리 오버헤드 vs 무상태 강제·환경 통제 상실(Code Jam 워밍업 해킹 사례); Google은 Borg가 대부분 이점을 커버하고 무상태 불가 앱(GCE/Spanner)이 많아 serverless에 크게 투자 안 함 — 단, 소규모 팀엔 매력적(공유 안 된 클러스터보다 저렴).
- Public vs Private: public cloud는 관리 오버헤드 아웃소싱·쉬운 스케일; lock-in 우려 완화책 — 오픈소스 아키텍처(Kubernetes) 위, 저수준 클라우드+고수준 오픈소스, multicloud(GKE+AKS), hybrid cloud(오버플로우).

---

## Summary (핵심 정리)
- 규모는 프로덕션 워크로드 실행을 위한 공통 인프라를 요구하며, 컴퓨트 솔루션은 소프트웨어에 표준화·안정적 추상화와 환경을 제공한다.
- 소프트웨어는 분산·관리 컴퓨트 환경(cattle 모델: 로컬 스토리지를 ephemeral로, hostname 하드코딩 회피)에 맞게 적응시켜야 한다.
- 조직의 컴퓨트 솔루션은 적절한 추상화 수준을 제공하도록 신중히 선택해야 한다 — AI 시대 대규모 워크로드 운영의 시니어 역량.
