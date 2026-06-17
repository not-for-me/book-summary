# 01. Trade-Offs in Data Systems Architecture

> Keywords: OLTP vs OLAP, data warehouse, data lake, ETL/ELT/reverse ETL, system of record vs derived data, cloud native, storage/compute separation, distributed vs single-node, microservices/serverless, data minimization, GDPR

## 챕터 개요 (3줄 요약)

- data system 설계에는 정답이 없고 오직 trade-off만 존재한다 — 모든 선택은 pros/cons를 갖는다.
- operational(OLTP) vs analytical(OLAP)의 분리에서 출발해, cloud vs self-hosting, distributed vs single-node라는 핵심 축을 비교한다.
- 기술적 요구뿐 아니라 law/society(GDPR, data minimization)까지 data system architecture를 규정한다.

---

## 1. Operational Versus Analytical Systems

> 데이터를 만드는 OLTP와 그 복사본을 읽기 전용으로 분석하는 OLAP은 access pattern·audience가 근본적으로 다르다.

### OLTP vs OLAP 대비
- Operational system(OLTP): backend service가 user action 기반으로 데이터를 read/write. point query(key로 소수 record 조회) 중심, 고정된 query가 application code에 내장.
- Analytical system(OLAP): operational 데이터의 read-only copy를 분석. 다수 record를 scan해 count/sum/average 등 aggregate. ad-hoc query 자유 (Tableau, Looker, Power BI).
- 데이터 표현: OLTP는 latest state(현재 시점), OLAP은 history of events(시간 누적).
- 규모: OLTP는 GB~TB, OLAP은 TB~PB.

```
OLTP: point query, fixed queries, current state, many small queries
OLAP: aggregate scan, ad-hoc queries, event history, few complex queries
```

### 역할(role)
- data engineer: operational <-> analytical 통합 및 전사 data infrastructure 책임.
- analytics engineer: 데이터를 model/transform 해 analyst·data scientist에게 유용하게 만듦.
- product analytics / real-time analytics (Pinot, Druid, ClickHouse): user-facing 제품에 내장된 low-latency analytical workload.

---

## 2. Data Warehousing

> OLTP에서 분석을 분리해 별도 read-only DB(data warehouse)로 옮기는 흐름. data silo, 성능 간섭, schema 부적합 문제를 해결.

### ETL / ELT
- data warehouse = 전사 OLTP 시스템들의 read-only copy. analyst가 OLTP 성능 영향 없이 자유롭게 query.
- ETL(extract–transform–load): 추출 → 분석 친화 schema로 변환·정제 → load. transform을 load 뒤에 하면 ELT.
- SaaS 소스(CRM, 결제 등)는 vendor API로만 접근 → Fivetran, Singer, Airbyte 같은 connector 사용.
- HTAP(hybrid transactional/analytical processing): 단일 system에서 OLTP+분석. 단 내부적으로는 OLTP+분석 system이 공통 interface 뒤에 결합된 형태가 많아 구분은 여전히 유효. data warehouse를 대체하지 않음.

```
OLTP systems --(periodic dump / continuous stream)--> ETL --> Data Warehouse
```

### From data warehouse to data lake
- data scientist는 relational/SQL보다 Pandas, scikit-learn, R, Spark 선호 (feature engineering, NLP, computer vision은 SQL로 표현 곤란).
- data lake: 어떤 file format/schema도 강제하지 않고 raw data를 그대로 저장하는 중앙 repository (Avro, Parquet, 이미지, sensor 등). object store 기반이라 저렴.
- ETL이 data pipeline으로 일반화. data lake가 operational→warehouse의 중간 정류장이 되기도 함.
- sushi principle: raw data is better — 소비자가 각자 필요에 맞게 raw를 변환.

### Beyond the data lake
- DataOps, governance, privacy(GDPR/CCPA) 부상.
- file 기반(주기적 재실행)을 넘어 stream of events(Chapter 12)로 초 단위 대응 (fraud/abuse 차단 등).
- reverse ETL: 분석 결과(예: 학습된 ML model)를 operational system으로 배포 (TFX, Kubeflow, MLflow).

---

## 3. Systems of Record and Derived Data

> 데이터 흐름을 명확히 하기 위한 구분: 권위 있는 원본(system of record) vs 그로부터 재생성 가능한 결과(derived data).

- System of record (source of truth): 권위 있는 canonical 버전. 새 데이터가 먼저 기록됨. 각 fact는 정확히 한 번(normalized) 표현. 불일치 시 이 값이 정답.
- Derived data system: 다른 system 데이터를 변환·가공한 결과. 잃어도 원본에서 재생성 가능 (cache, index, materialized view, denormalized value, trained model).
- derived data는 redundant 하지만 read 성능에 필수. 동일 source에서 여러 view를 derive 가능.
- 도구 자체가 system of record/derived를 결정하지 않음 — 사용 방식이 결정. derived 갱신을 위해 data pipeline(Chapter 11)로 data integration 필요.

---

## 4. Cloud Versus Self-Hosting

> build vs buy, 누가 만들고 누가 운영하는가의 spectrum. core competency는 in-house, 비핵심은 vendor.

### Pros/Cons of cloud services
- 부하가 예측 가능하고 운영 역량이 있으면 self-host가 더 저렴할 수 있음.
- 모르는 system은 cloud가 더 빠르고 쉬움 (인력 채용·교육 비용 회피).
- 가변 load(특히 analytical)에 cloud가 유리 — 미사용 자원 반납으로 비용 절감.
- 단점: control 부재(기능 추가 불가, 장애 시 대기, 내부 metric/log 접근 불가), vendor lock-in, 가격/제품 변경 리스크, 지정학적 sanction, data 보안·규제 준수 복잡성.

### Cloud Native System Architecture
- cloud native: cloud service를 활용하도록 처음부터 설계된 architecture. 동일 hardware에서 더 나은 성능, 빠른 복구, 빠른 scaling, 더 큰 dataset.
- 예: OLTP는 AWS Aurora, Azure SQL DB Hyperscale, Google Cloud Spanner / OLAP는 Snowflake, BigQuery, Azure Synapse.
- layering: object storage(S3, Azure Blob, R2) 위에 상위 service 구축 (Snowflake가 S3 의존).

### Separation of storage and compute
- cloud의 local disk는 ephemeral cache처럼 취급(instance 실패/교체 시 소실).
- virtual disk(EBS 등)는 별도 machine이 block device를 emulate — 모든 I/O가 network call이라 glitch에 민감.
- cloud native는 virtual disk 회피, object storage 같은 전용 storage service 사용. storage와 compute를 disaggregated — S3는 저장만, 분석은 외부에서 실행 → network 전송 발생.
- multitenant: 여러 customer가 같은 hardware 공유 → 활용도 상승 하지만 isolation 엔지니어링 필수.

```
Traditional: [ CPU + RAM + Disk ] on same machine
Cloud Native: [ Compute nodes ] <--network--> [ Object Storage (S3) ]
```

### Operations in the Cloud Era
- DBA/sysadmin → DevOps/SRE(Google). 개별 machine 관리에서 service 관리로 이동.
- 강조점: automation, ephemeral VM, frequent updates, learning from incidents, 조직 지식 보존.
- capacity planning → financial planning, performance optimization → cost optimization. quota/limit 사전 인지 필요. 보안·통합·monitoring은 outsourcing 불가.

---

## 5. Distributed Versus Single-Node Systems

> network로 통신하는 여러 node = distributed system. 불가피한 경우가 많지만 가능하면 single-node 유지가 단순·저렴.

### 분산을 택하는 이유
- inherent distribution(여러 user/device), cloud service 간 요청, fault tolerance/HA, scalability, latency(지역 근접), elasticity, 특화 hardware(GPU 등), 법적 data residency, sustainability(재생에너지 시점/위치).

### Problems with Distributed Systems
- 모든 network 호출은 실패(timeout) 가능 → retry 안전성 불확실(Chapter 9).
- 원격 호출은 동일 process 함수 호출보다 훨씬 느림 — 대용량은 computation을 데이터 쪽으로 이동이 유리. node가 많다고 항상 빠르지 않음(COST).
- troubleshooting 어려움 → observability (OpenTelemetry, Zipkin, Jaeger의 tracing).
- service별 DB 분리 시 cross-service consistency가 application 문제. distributed transaction은 microservices에서 거의 안 씀.
- 단일 머신 발전(DuckDB, SQLite, KùzuDB)으로 많은 workload가 single-node로 가능.

### Microservices and Serverless
- SOA → microservices: 하나의 명확한 목적, network API 노출, 팀별 독립 책임. 독립 배포·독립 자원 할당·구현 은닉 장점. service마다 별도 DB(공유 DB는 사실상 API화되어 변경 곤란).
- 단점: 테스트 복잡, 배포·monitoring 인프라 필요(Kubernetes), API 진화 어려움(OpenAPI, gRPC). 본질은 사람 문제(팀 독립성)의 기술적 해법 — 작은 조직엔 과한 overhead.
- Serverless/FaaS: 인프라 관리를 vendor에 위임, 요청 기반 자동 자원 할당, 실행 시간만 과금. 단 실행 시간 제한, runtime 제약, cold start.

### Cloud Computing vs Supercomputing (HPC)
- HPC: 과학 계산(기상, 기후, molecular dynamics), checkpoint 후 실패 시 전체 중단·재시작, shared memory/RDMA, 특화 network topology(mesh/torus), node 근접 가정.
- Cloud: 지속 가용 online service, 상호 불신 조직 공유 → VM isolation/암호화/인증, IP/Ethernet Clos topology, 다지역 분산.

---

## 6. Data Systems, Law, and Society

> architecture는 기술 요구뿐 아니라 privacy 규제와 사회적 책임으로도 규정된다.

- GDPR(2018~), CCPA, EU AI Act가 personal data 사용·권리를 규율. 자동화 의사결정(대출, 보험, 채용, 범죄 의심)의 사회적 영향에 대한 책임.
- right to be forgotten: 삭제 요청 권리 vs append-only log/immutable 설계, derived dataset(ML training data) 삭제라는 엔지니어링 난제.
- 규제는 특정 기술을 강제하지 않고 high-level 원칙만 제시 → 해석·구현은 엔지니어 몫.
- 저장 비용은 S3 청구서뿐 아니라 leak/유출 책임, 법적 fine 리스크 포함. 범죄화된 행위(예: 위치 데이터)를 저장하면 user 안전 위협.
- data minimization(Datensparsamkeit): big data 사상과 반대로, 특정·명시된 목적에만 수집·보관. PCI, SOC Type 2 같은 compliance/감사도 영향.

---

## Summary (핵심 정리)

- 핵심 테마는 trade-off — data system의 주요 결정에는 단일 정답이 없고 각 선택지가 pros/cons를 가진다.
- 4개 축: operational(OLTP) vs analytical(OLAP) / cloud vs self-hosting / distributed vs single-node / business vs user rights(law). 그리고 data warehouse·data lake·ETL, system of record vs derived data, storage-compute 분리라는 핵심 개념을 도입.
- 다음 연결: 내부 data layout 차이는 Chapter 4(Storage and Retrieval), 분산 시스템 난제는 Chapter 9, 법/윤리는 Chapter 14에서 심화.