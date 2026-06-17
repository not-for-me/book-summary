# 05. Encoding and Evolution

> Keywords: encoding/decoding(serialization), backward/forward compatibility, rolling upgrade, JSON/XML/CSV, JSON Schema, MessagePack, Protocol Buffers/Thrift(field tag), Avro(writer/reader schema), schema evolution, dataflow(database/REST/RPC/message), service discovery/load balancing/service mesh, durable execution/workflow, message broker, actor model

## 챕터 개요 (3줄 요약)
- 애플리케이션은 변하므로 data format도 진화한다 — old/new 코드와 데이터가 공존하려면 backward(신코드가 구데이터 읽기)·forward(구코드가 신데이터 읽기) compatibility가 필수.
- in-memory 객체를 byte로 바꾸는 encoding 포맷(JSON, Protocol Buffers, Avro)을 schema evolution 관점에서 비교한다.
- 데이터는 database·service(REST/RPC)·event-driven(message broker/actor) 등 다양한 경로로 흐르며, 각 경로에서 compatibility가 rolling upgrade와 evolvability를 가능케 한다.

---

## 1. Compatibility 기본
> rolling upgrade로 old/new 코드가 동시에 돌기에 양방향 compatibility가 필요하다.

- `rolling upgrade`(staged rollout): 일부 node씩 새 버전 배포 → 무중단·잦은 릴리스·evolvability. client-side는 사용자가 업데이트 안 할 수도 있음.
- `backward compatibility`: 신코드가 구데이터 읽기(보통 쉬움 — 구포맷을 알고 있음).
- `forward compatibility`: 구코드가 신데이터 읽기(어려움 — 모르는 필드를 무시·보존해야 함). 모르는 필드를 보존하지 않으면 update-rewrite 시 데이터 손실(Figure 5-1).
- API: 구client→신service는 request의 backward + response의 forward 필요(반대도 대칭).

---

## 2. Formats for Encoding Data
> in-memory ↔ byte 변환(encoding/decoding). 언어 내장 포맷은 지양, 표준 포맷 사용.

### Language-specific & Textual
- 언어 내장(Java Serializable, pickle, Marshal): 언어 종속, 보안 취약(임의 클래스 instantiate → RCE), versioning·효율 부실 → transient 외 비권장.
- JSON/XML/CSV: 널리 지원·human-readable이지만 문제 — 숫자 모호(JSON은 int/float 미구분, 2^53 초과 부정확: X의 post ID 이중 표기), binary string 미지원(Base64로 우회, +33%), CSV는 schema 없음.
- `JSON Schema`: OpenAPI·schema registry·DB validator에 널리 쓰임. open(additionalProperties=true, 기본)/closed content model, validation(min/max 등). 강력하나 복잡(remote schema, 조건부 규칙, evolution 어려움).

### Binary: MessagePack
- JSON의 binary 변형(MessagePack, CBOR, BSON). schema 미규정이라 field name을 데이터에 포함 → 절약 미미(81→66 byte).

### Protocol Buffers (& Thrift)
- schema 필수(IDL). code generation으로 각 언어 클래스 생성. 예제 33 byte.
- field name 대신 `field tag`(숫자) 사용 + type/tag를 1 byte에 packing + variable-length integer.
- schema evolution: field name은 변경 가능(데이터는 name 미참조)하나 tag는 불변. 새 field는 새 tag로 추가(구코드는 모르는 tag 무시 → forward 유지, type annotation으로 skip 길이 계산). 새 field에 default로 backward 유지. field 제거 시 tag 재사용 금지(reserved). datatype 변경은 truncation 위험.

### Avro
- Hadoop용으로 시작. IDL(사람용)+JSON(기계용) schema. tag number 없음. 예제 32 byte(최소).
- 데이터에 field/type 식별자 없이 값만 concatenation → `writer's schema`와 `reader's schema`로 디코딩.
- schema resolution: field를 name으로 매칭. writer에만 있는 field는 무시, reader에만 있으면 default로 채움.
- evolution: default 있는 field만 추가/제거 가능. forward=writer가 신schema, backward=writer가 구schema. null은 union type(`union{null,long}`)으로 명시. field rename은 alias(backward only).
- writer's schema 전달: 대용량 파일은 시작에 1회(object container file), DB는 record마다 version 번호+schema registry(Confluent), network는 연결 시 협상(Avro RPC).
- 장점: `dynamically generated schema`에 친화적(relational schema→Avro schema 자동 생성, 컬럼 변경 시 재생성만). Protocol Buffers는 tag를 수동 관리해야 함.

### Merits of Schemas
- binary schema 포맷 장점: field name 생략으로 compact, schema가 최신 문서 역할, 배포 전 compatibility 검사, 정적 언어에서 code generation·compile-time type check.
- ASN.1(1984, SSL 인증서 DER)과 유사하나 복잡. schema evolution은 schemaless의 유연성 + 더 나은 보장·tooling 제공.

---

## 3. Modes of Dataflow
> 데이터가 누가 encode하고 누가 decode하는가 — database/service/message 경로별 compatibility.

### Through Databases
- writer가 encode, reader가 decode. 동시 접근 + rolling upgrade로 forward도 필요(신버전이 쓴 걸 구버전이 읽음, 모르는 필드 보존 주의).
- `data outlives code`: 5년 전 데이터가 원래 인코딩으로 잔존. migration은 비싸 비동기·best-effort(LSM compaction 시 재기록, ALTER+null default). 복잡한 변경은 app 레벨 rewrite.
- archival: 스냅샷은 최신 schema로 일관 인코딩 → Avro object container file, 분석용 Parquet 적합.

### Through Services: REST and RPC
- service = 서버가 노출하는 API. DB와 달리 business logic이 정한 input/output만 허용(encapsulation). 독립 배포·evolvability가 목표.
- `web service`(HTTP 기반): client→service, service→service, 조직 간. `REST`(URL로 resource 식별, HTTP 기능 활용) → RESTful. IDL: OpenAPI/Swagger(JSON), Protocol Buffers(gRPC). framework: Spring Boot, FastAPI, gRPC.
- `RPC`의 결함: 네트워크 호출을 로컬 함수처럼 보이게(location transparency) 하지만 근본적으로 다름 — 예측 불가·timeout(결과 모름)·retry 시 중복(idempotence 필요)·가변 latency·인자 인코딩·언어 간 타입 변환. REST는 state transfer를 함수 호출과 구분.
- `service discovery`/load balancing: hardware/software LB(NGINX, HAProxy), DNS(전파 느림), 레지스트리(etcd, ZooKeeper, heartbeat), `service mesh`(Istio, Linkerd — sidecar, 로컬 연결로 TLS·observability).
- RPC evolution: 서버 먼저·client 나중 가정 → request backward + response forward. 조직 간이라 호환을 오래 유지(다중 버전 병행). API versioning(URL/Accept header).

### Durable Execution and Workflows
- service 여러 개를 잇는 일련의 step = `workflow`, 각 step = `task`(activity/durable function). `workflow engine`(orchestrator 스케줄 + executor 실행). Airflow/Dagster(ETL), Camunda/Orkes(BPMN), Temporal/Restate(durable execution).
- `durable execution`: workflow에 exactly-once 제공. task 실패 시 재실행하되 성공한 RPC/state 변경은 skip(WAL에 로깅 후 이전 결과 반환).
- 주의: 외부 service는 idempotent API 필요, RPC 순서·결정성 유지(코드 reorder·random·clock 금지, 새 버전 별도 배포).

### Event-Driven Architectures
- request = event/message. RPC와 달리 sender가 응답 대기 안 함, `message broker`(event broker/queue) 경유(임시 저장).
- 장점: buffer(과부하 흡수), crash 시 재전송, service discovery 불필요, 다중 수신자, sender-recipient 분리(asynchronous).
- broker: RabbitMQ, NATS, Redpanda, Kafka, 클라우드(Kinesis, Pub/Sub). 패턴: queue(소비자 1명) vs topic(구독자 전원). data model 무관(Protocol Buffers/Avro/JSON + schema registry). 일부는 무기한 저장(event sourcing). 재발행 시 모르는 필드 보존 주의.
- `actor model`: 동시성 모델 — thread 대신 actor(로컬 state, async message). distributed actor framework(Akka, Orleans, Erlang/OTP)는 다중 node로 확장. message 손실 가정이라 location transparency가 RPC보다 자연스러움. rolling upgrade 시 여전히 compatibility 필요.

---

## Summary (핵심 정리)
- rolling upgrade로 서로 다른 버전이 공존하므로 모든 dataflow는 backward(신코드가 구데이터)·forward(구코드가 신데이터) compatibility를 가져야 evolvability를 확보한다.
- 인코딩: 언어 내장(비권장), 텍스트(JSON/XML/CSV, 타입 모호), binary schema-driven(Protocol Buffers의 field tag, Avro의 writer/reader schema)는 compact·명확한 compatibility·code generation 제공.
- dataflow 경로: database(writer encode/reader decode), REST·RPC(client↔server), event-driven(message broker/actor). 각 경로에서 약간의 주의로 무중단 진화가 가능.
- 다음 연결: Ch6에서 replication(데이터를 여러 node에 복제)을 다룸.