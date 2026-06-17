# 11. Batch Processing

> Keywords: online vs offline(batch) system, immutable input/derived output, human fault tolerance/time travel, Unix pipeline(sort/uniq), sort vs in-memory aggregation, distributed filesystem(HDFS)/object store(S3), job orchestrator(YARN/Kubernetes)/scheduler/resource allocation, workflow/DAG(Airflow), MapReduce(mapper/reducer), dataflow engine(Spark/Flink), shuffle/sort-merge join/secondary sort, SQL/DataFrame, ETL/analytics/ML/serving derived data

## 챕터 개요 (3줄 요약)
- batch processing은 read-only·immutable input에서 매번 output을 새로 생성하는 offline 작업으로, 성능 지표는 throughput이다.
- input immutability 덕분에 버그 시 rollback·rerun이 가능(human fault tolerance, time travel) — 분산 batch framework는 distributed OS처럼 filesystem·scheduler·program을 가진다.
- MapReduce에서 출발해 dataflow engine(Spark/Flink)로 진화했고, 핵심은 shuffle(분산 정렬)이며 SQL/DataFrame으로 usability가 향상되었다.

---

## 1. Batch Processing 개념 & Unix Tools
> immutable input → derived output. rollback·rerun이 핵심 이점.

- `online system`(요청-응답, response time 중심) vs `offline/batch system`(대량 처리, throughput 중심).
- batch는 input read-only, output 매번 재생성 → 버그 시 코드 rollback+rerun이면 output 복구(`human fault tolerance`, object store/table format의 `time travel`). read/write transaction은 이 속성 없음(irreversibility 최소화 = Agile 친화).
- 한계: 전체 job 완료 후에야 다음 job, input 1 byte 변경에도 전체 재처리. 대안은 stream processing(Ch12).
- MapReduce(Google 2004)가 영향 — 지금은 obsolete, Spark/Flink·cloud DW로 이동. 저장은 HDFS→S3.

### Unix Tools
- log 분석 예: `cat | awk | sort | uniq -c | sort -rn | head`. gigabyte를 수초에 처리, powerful·composable.
- `sort vs in-memory aggregation`: in-memory hash table는 distinct key가 메모리에 들어갈 때만. working set이 메모리 초과면 sort가 유리(disk spill+mergesort, sequential access). GNU sort는 자동 disk spill·병렬화. 단 단일 머신 한계 → 분산 framework 필요.

---

## 2. Batch Processing in Distributed Systems
> 분산 batch framework = distributed OS(filesystem + scheduler + program).

### Distributed Filesystem & Object Store
- DFS(HDFS): 파일을 큰 block(128MB)으로 분할·여러 머신 복제(또는 erasure coding). data node가 block 제공, NameNode가 metadata. shared-nothing(NAS/SAN의 shared-disk와 대비). 계산을 데이터 있는 머신에서 실행(network 절약).
- `object store`(S3): URL(bucket+key), get/put, 객체 immutable(update는 전체 재기록), directory 없음(prefix list = recursive ls), rename 비원자적. storage/compute 분리(독립 scaling). S3 API가 사실상 표준(MinIO, R2).

### Job Orchestration
- orchestrator(YARN, Kubernetes)는 분산 OS kernel 역할. 구성: `task executor`(NodeManager/kubelet, cgroup isolation), `resource manager`(cluster 상태, ZooKeeper/etcd), `scheduler`(어느 node에 task 배치).
- `resource allocation`: fairness vs efficiency 균형. gang scheduling(전량 확보 후 시작, idle 위험), preemption(efficiency↓), starvation 위험. NP-hard → heuristic(FIFO, DRF, priority, bin-packing).
- `workflow`/`DAG`: job 출력이 다른 job 입력. 보통 DFS/object store 경유로 decouple. workflow scheduler(Airflow, Dagster, Prefect)가 의존성 관리(50~100 job 흔함).
- fault: 장기 실행이라 task 실패 흔함. preemption(spot instance로 비용↓). batch는 output 재생성이라 task 단위 재시도 쉬움. MapReduce는 중간 데이터를 DFS에 기록, Spark는 메모리(lineage로 재계산), Flink는 checkpoint.

---

## 3. Batch Processing Models
> MapReduce → dataflow engine. 핵심 알고리즘은 shuffle.

### MapReduce
- 4단계: 파일→record, `mapper`(record→key-value, 독립·병렬), 정렬(암묵적), `reducer`(같은 key 값 집계). 함수형(map/reduce, mutable state 회피로 병렬화).
- 단점: join을 직접 구현, file 기반 I/O로 느림(pipelining 불가).

### Dataflow Engines (Spark/Flink)
- 전체 workflow를 하나의 job으로 모델링(`dataflow engine`). map/reduce 교대 강제 없이 유연한 operator(join/group by/filter). 장점: 불필요한 sort 회피, operator 결합(데이터 복사↓), locality 최적화, 중간 상태 메모리/로컬 디스크, operator 즉시 시작, process 재사용. MapReduce보다 빠름.

### Shuffle (분산 정렬, 랜덤 아님)
- batch processor의 foundational 알고리즘(join·aggregation). mapper가 reducer별 파일 생성(key hash로 분배)+로컬 정렬 → reducer가 복사·mergesort 병합 → key별 reducer 호출.
- `sort-merge join`: user activity(fact)와 user DB(dimension)를 user ID key로 shuffle → reducer가 한 user의 모든 record를 메모리 하나로 처리(network 요청 불필요). `secondary sort`로 DB record를 먼저 보게 정렬.

### Query Languages & DataFrames
- SQL이 lingua franca(legacy DW·도구 호환·interactive 탐색). cost-based optimizer(Hive/Trino/Spark/Flink)가 join 알고리즘·순서 최적화.
- batch와 cloud DW 수렴: batch는 SQL·Parquet·vectorization 채택, DW는 cloud·scheduling·shuffle 채택. 단 graph/ML/multimodal은 SQL로 어려움, row 단위는 columnar에 비효율.
- `DataFrame` API(R/Pandas→Spark/Flink/Daft): 관계 연산 함수 호출. Pandas는 즉시 실행, Spark는 query plan+최적화 후 분산 실행. 분산 DataFrame은 보통 index/order 없음(성능 주의).

---

## 4. Batch Use Cases
> 대량 데이터+freshness 비중요한 곳. 의외로 많은 작업이 이 모델에 적합.

- `ETL`: 추출·변환·load. embarrassingly parallel(filter/projection). robust scheduler·troubleshooting·rerun. data mesh/data contract/data fabric으로 팀별 pipeline 기여.
- `analytics`(OLAP): query engine이 DFS/object store 읽기, table format(Iceberg)+catalog(Unity) → `data lakehouse`. pre-aggregation(OLAP cube, Druid/Pinot) vs ad-hoc query. Tableau/Power BI/Superset 연동.
- `ML`: feature engineering, model training, batch inference. Spark MLlib/Flink FlinkML. graph는 `BSP`/Pregel model(Giraph, GraphX, Gelly). LLM 전처리(text 추출·중복 제거·tokenize/embedding, Ray/Kubeflow/Flyte). notebook(Jupyter/Hex).
- `serving derived data`: 추천·리포트·ML feature를 production DB로. DB에 직접 record별 write는 나쁨(느림, 과부하, all-or-nothing 깨짐). 더 나음 — Kafka stream으로 push(sequential write, buffer, 다수 consumer, DMZ 경계) 또는 batch에서 DB 통째 build 후 bulk-import(원자적 swap, 단 incremental 어려움 → hybrid).

---

## Summary (핵심 정리)
- batch processing은 immutable·bounded input에서 output을 생성 — rerun·debugging이 side effect 없이 가능. 3 구성: orchestration(언제/어디서), storage(DFS/object store), computation(처리).
- DFS/object store는 block 복제·caching·metadata service로 대용량 파일 관리, pluggable API로 framework와 연동. orchestrator는 task 스케줄·resource 할당·fault 처리, workflow orchestrator는 의존성 graph 관리.
- model: MapReduce(map/reduce) → dataflow engine(Spark/Flink, 더 빠르고 쉬움). shuffle이 grouping·join·aggregation을 가능케 하는 foundational 연산. SQL/DataFrame으로 usability 향상.
- use case: ETL pipeline, analytics(pre-agg/ad-hoc), ML(training data 준비), serving derived data(stream/bulk-load).
- 다음 연결: Ch12에서 unbounded stream을 다루는 stream processing — job이 끝나지 않는다.