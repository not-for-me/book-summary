# 11. Clusters and Job Queues

## 챕터 개요 (3줄 요약)

- 클러스터(cluster)는 여러 컴퓨터가 협력해 작업을 푸는 것으로, 확장성·신뢰성·동적 스케일링을 주지만 시스템 관리·지연·동기화라는 큰 비용이 따른다.
- 클러스터로 가기 전에 프로파일링·컴파일·단일 머신 멀티코어·RAM 절감을 먼저 활용해야 하며, 단순함이 클러스터 설계의 핵심 원칙이다.
- IPython Parallel(연구용), 메시지 브로커(Queue/pub-sub, 프로덕션용), Docker(재현 가능한 환경)가 주요 클러스터링 도구다.

---

## 1. Benefits & Drawbacks (장점과 단점)

> 클러스터는 머신을 추가해 쉽게 확장하고 신뢰성을 높이며 동적 스케일링이 가능하지만, 직렬→병렬을 넘어선 사고 전환과 시스템 관리 부담을 요구한다.

- 장점: 머신 추가로 컴퓨팅 확장, 부품 고장에도 동작하는 신뢰성, 수요에 따른 동적 스케일링(비용 효율적), 지리적 분산·이종 소프트웨어 환경의 견고성.
- 단점: 머신 간 지연(latency), 버전 동기화, 시스템 관리가 가장 큰 도전이며, 추가 계획이 핵심 작업에서 주의를 분산시킨다.
- 머신마다 설정·부하·로컬 데이터가 달라, 데이터를 어떻게 전달하고 프로세스/머신 실패 시 무엇이 일어나는지 고려해야 한다.
- 일부 실패는 허용 가능(콘텐츠 웹 서비스)하지만 고빈도 거래 시스템에선 치명적이며, 고정 인프라 유지비(시스템 엔지니어 등)가 비싸다.
- 문서화된 재시작 계획이 없으면 최악의 시점에 작성하게 되며, 콜드 스타트로 백로그가 쌓일 수 있다.
- 교훈 사례: Knight Capital($4.62억 손실, 불완전한 업그레이드·리뷰 부재), Skype(24시간 장애, 연쇄 실패) — 단위 테스트·체크리스트·이종 환경의 중요성.

---

## 2. Cluster Design & Best Practices (클러스터 설계와 모범 사례)

> 보통 로컬 ad hoc 클러스터로 시작하며, 작업 큐(queue)가 가장 흔하고 이해하기 쉬운 소프트웨어 아키텍처다.

- 하드웨어: 사내 클러스터(유지보수 인력 필요), 클라우드(EC2/Azure, 하드웨어 지원 위임), 커스텀(InfiniBand·RAID·GPU), 분산형(SETI@home/BOINC).
- 소프트웨어: 작업 큐(가장 흔함), 메시지 패싱(메시지 버스), IPC(전문가용, 잘못 설정하기 쉬움).
- 시작법: 한 머신에 job server + processor 하나로 시작해 CPU 바운드면 CPU당 하나, I/O 바운드면 여러 개를 띄우고, 1/0·kill -9·전원 차단으로 견고성을 테스트한다.
- 부팅 시 cron·Circus·supervisord로 컴포넌트를 안정적으로 시작하고, Chaos Monkey 같은 도구로 복원력을 검증한다.
- 고통 회피: 빠른 시스템보다 디버깅 쉬운 시스템, JSON 같은 사람이 읽을 수 있는 메시지, Fabric/Salt/Chef/Puppet 등 배포 시스템, 긍정 보고(매일 상태 이메일)·Ganglia 모니터링.

---

## 3. IPython Parallel (연구용 클러스터)

> IPython Parallel은 로컬·원격 처리 엔진에 데이터를 푸시하고 작업을 분배하며, ZeroMQ 미들웨어로 Jupyter Notebook과 같은 메커니즘을 쓴다.

- 4개 구성요소: engine(코드 실행 인터프리터), controller(작업 분배 인터페이스), hub(엔진·스케줄러·클라이언트 추적), scheduler(비동기 인터페이스).
- `ipcluster start -n 4`로 엔진을 띄우고 `ipp.Client()`로 연결하며, `c[:].apply_sync(...)`로 모든 엔진에 함수를 실행한다.
- `sync_imports` 컨텍스트 매니저로 로컬·원격 동시 import하고, `push`/`pull`로 데이터를 주고받는다.
- 4개 로컬 엔진으로 파이를 추정하면 약 47초로 멀티프로세싱과 유사하다.
- MPI 지원, Joblib·Dask 백엔드로 사용 가능하며, ElastiCluster로 AWS·Azure·OpenStack 같은 큰 클러스터로 확장할 수 있다(ZeroMQ는 기본 보안 없음, 필요 시 SSH).

### IPython Parallel (원문 의도 유지, 새 예제)

```python
import ipyparallel as ipp

client = ipp.Client()
dview = client[:]                      # direct view (전체 엔진)

with dview.sync_imports():             # 로컬+원격 동시 import
    import random

# 모든 엔진에 데이터 푸시
dview.push({'config': {'trials': 1_000_000}})

# 각 엔진에서 함수 실행 (블로킹)
results = dview.apply_sync(lambda: sum(random.random() < 0.5
                                       for _ in range(config['trials'])))
```

---

## 4. Message Brokering — Queue & Pub/Sub (메시지 브로커)

> 프로덕션에선 노드 장애·네트워크 다운에 견디는 메시지 브로커가 필요하며, Queue(생산/소비 불균형 버퍼)와 pub/sub(토픽 구독)이 핵심 개념이다.

- 주요 브로커: ActiveMQ, RabbitMQ, Kafka, ZeroMQ(직접 운영) / Google Pub/Sub, Amazon MQ(관리형).
- Queue는 메시지 버퍼로, 생산이 소비보다 빠르면 소비자를 수평 확장(horizontal scaling)하고, 소비자가 죽어도 메시지가 큐에 남아 전달을 보장한다.
- Queue 고려사항: 메모리 초과·노드 오프라인 시 동작, 분산·중복성, 소비자의 처리 완료 확인(ack), 메시지 만료·순서.
- pub/sub은 큐의 모음으로, publisher가 토픽에 데이터를 올리면 모든 subscriber가 동일 사본을 받고, 각 subscriber의 consumer 중 하나만 그 메시지를 처리한다(자동 부하 분산).
- publisher/subscriber가 decoupled되어 네트워크 변화에도 견고하며, consumer는 한 토픽을 소비·변환해 다른 토픽에 발행하는 체인을 만든다.
- pub/sub 고려사항: 메시지 보장(at least/at most once), 토픽·consumer 수준 보장, 아카이빙·리플레이, 단일 실패 지점.

### Queue vs Pub/Sub (개념도)

```
  [Queue]   producer -> [ msg msg msg ] -> one consumer takes each
            (소비자 추가로 수평 확장, 처리 완료까지 메시지 보존)

  [Pub/Sub] publisher -> topic --+--> subscriber A (consumer 1 of N)
                                 +--> subscriber B (consumer 1 of M)
            (각 subscriber는 사본 1개, 그 안의 consumer 중 하나가 처리)
```

---

## 5. Docker (재현 가능한 환경)

> Docker는 재현 가능한 런타임 환경을 만들어 팀 간 코드 공유와 클러스터 배포를 쉽게 하며, OS 레벨 가상화라 CPU/메모리 성능 저하가 거의 없다.

- 다른 클러스터링 도구: ZeroMQ(저수준 고성능), Celery(분산 태스크 큐), Airflow/Luigi(DAG 작업 체인), Amazon SQS.
- Docker는 cgroups로 호스트 OS 내 격리된 namespace에서 코드를 실행해 하드웨어 가상화(VMware 등)와 달리 오버헤드가 거의 없다(단 macOS/Windows는 Linux 커널을 가상화).
- Dockerfile은 베이스 이미지 선택, WORKDIR 설정, requirements 복사·설치, 전체 복사, CMD 실행 순으로 작성한다.
- requirements를 먼저 복사하고 나중에 전체를 복사하면, 소스만 바뀌었을 때 pip install 캐시를 재사용한다.
- 성능 주의점: 빌드 컨텍스트 과대(.dockerignore), 파일시스템 레이어(host mount·read-only volume), 가상 네트워크 오버헤드, GPU는 nvidia-docker/--device.
- 장점: 이전 버전 재현·회귀 테스트, 컨테이너 레지스트리(docker pull/push)로 팀 공유, kubernetes와 결합해 클러스터 노드 환경 호환성 문제 해소.

### Dockerfile (원문 의도 유지, 새 예제)

```dockerfile
FROM python:3.12-slim
WORKDIR /usr/src/app
# requirements 먼저 복사 → 소스만 바뀌면 pip 캐시 재사용
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "./compute.py"]   # JSON 배열로 시그널 안전성 확보
```

---

## Summary (핵심 정리)

- 클러스터는 확장성·신뢰성·동적 스케일링을 주지만 시스템 관리·지연·동기화 비용이 크므로, 단일 머신(수직 확장)으로 해결 가능하면 그것이 더 쉽고, 클러스터로 갈 땐 단순함과 디버깅 용이성을 우선해야 한다.
- 연구에는 IPython Parallel(로컬·원격 엔진에 작업 분배, ElastiCluster로 클라우드 확장)이 자연스럽고, 프로덕션에는 노드 장애에 견디는 메시지 브로커(Queue로 수평 확장, pub/sub로 자동 부하 분산)가 필요하다.
- Docker는 cgroups 기반 OS 가상화로 CPU/메모리 오버헤드가 거의 없으면서 재현 가능한 환경·팀 공유·클러스터 배포(kubernetes)를 가능하게 하므로, 클러스터의 환경 호환성 문제를 근본적으로 해소한다.
