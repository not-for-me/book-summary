# 08. A Sample Implementation

## 챕터 개요 (3줄 요약)

- YAML 기반 커스텀 인터페이스로 데이터 컨트랙트를 정의하고, 이를 토대로 컨트랙트 주도 아키텍처를 실습으로 구현한다.
- 하나의 컨트랙트가 BigQuery 테이블, 생성자용 라이브러리, 스키마 레지스트리, 익명화 서비스 네 가지를 구동한다.
- IaC(Infrastructure as Code) 툴 Pulumi와 JSON Schema, Confluent Schema Registry를 활용해 리소스 생성·스키마 진화·툴링을 다룬다.

---

## 1. Creating a Data Contract

> 데이터 생성자가 작성하는 YAML 기반 명세를 정의하고, 기계 판독 가능성을 검증해 컨트랙트 주도 아키텍처의 토대로 삼는다.

- 컨트랙트는 BigQuery 테이블, 코드 라이브러리, 스키마 레지스트리, 익명화 서비스를 구동하는 단일 토대다.
- 구현 방식은 Avro/protobuf, YAML/Jsonnet, Python/TypeScript 등 다양하나 본 예제는 커스텀 YAML을 사용한다.
- YAML은 사람이 읽기 쉽고 기계 판독 가능하며, 필요한 메타데이터(스키마, 검증 규칙, 익명화 전략)를 유연하게 담는다.
- Python의 yaml 라이브러리로 컨트랙트를 파싱하고 name·owner 등 메타데이터를 추출한다.
- 검증 코드로 모든 컨트랙트에 owner가 있는지 등을 강제하고, CI(Continuous Integration) 체크로 빌드를 실패시킬 수 있다.

```
name: Customer
owner: product-team@data-contracts.com
version: 1
fields:
  email: { type: string, pattern: "...", anonymization_strategy: email }
```

---

## 2. Providing the Interfaces to the Data (BigQuery + IaC)

> 컨트랙트를 BigQuery JSON 스키마로 변환하고 IaC 툴 Pulumi로 테이블을 동적 프로비저닝·동기화한다.

- 데이터 인터페이스로 Google BigQuery 테이블을 컨트랙트의 스키마로부터 생성한다.
- IaC는 코드로 인프라를 프로비저닝하며 일관성·재사용성·테스트성·버전관리(Git) 이점을 가진다(예: Terraform, Ansible).
- 본 예제는 Python으로 인프라를 정의할 수 있는 Pulumi를 사용한다.
- DataContract 클래스가 YAML 컨트랙트를 BigQuery용 커스텀 JSON 스키마로 변환한다.
- `pulumi up`으로 데이터셋·테이블을 생성하고, 컨트랙트 변경 시 `pulumi up` 재실행으로 스키마를 동기화한다.
- 같은 패턴을 Snowflake, Amazon Redshift, Pub/Sub 스트리밍 토픽 등 어떤 리소스에도 적용할 수 있다.

```
data_contract = DataContract("../contracts/Customer.yaml")
customerTable = bigquery.Table(..., schema=data_contract.bigquery_schema())
```

---

## 3. Creating Libraries for Data Generators

> 컨트랙트를 JSON Schema로 변환해 기존 오픈소스 라이브러리로 데이터 검증 클라이언트 라이브러리를 만든다.

- 클라이언트 라이브러리는 데이터 변환·검증·커스텀 로직으로 서비스 간 데이터 일관성을 돕고 소비자의 역직렬화도 지원한다.
- 컨트랙트의 fields를 순회해 properties·required·enum·pattern을 추출, JSON Schema를 생성한다.
- Python의 jsonschema 라이브러리는 한 줄 `validate(event, schema)`로 데이터를 검증한다.
- 검증 실패 시(필수 필드 누락, pattern 불일치, enum 위반) 오류를 반환해 다운스트림 영향을 차단한다.
- 오픈 포맷 변환 덕분에 생성자가 개발·운영 양쪽에서 빠르게 검증 라이브러리를 활용할 수 있다.

---

## 4. Populating a Central Schema Registry

> 컨트랙트를 JSON Schema로 변환해 스키마 레지스트리에 등록하고, 다중 버전 저장·호환성 검사로 스키마 진화를 관리한다.

### Registering & retrieving

- Confluent Schema Registry는 JSON Schema를 네이티브 지원하며 Docker로 로컬 실행한다(Kafka, ZooKeeper 의존).
- 각 스키마는 고유 ID와 subject(이름)를 가지며 ID·subject·version·latest로 조회 가능하다.

### Managing schema evolution

- 비파괴 변경(선택적 country 필드 추가)은 기존 소비자에 영향 없이 낮은 마찰로 새 버전(v2)으로 등록된다.
- 파괴 변경(email 필드 제거)은 레지스트리의 호환성 검사에 걸려 등록이 거부되고 오류(HTTP 409)를 반환한다.
- 이 호환성 검사를 CI 체크로 활용해 운영 배포 전 파괴 변경을 막는다.
- 파괴 변경 시 마이그레이션 계획을 세우고, 새 subject(예: Customer.v2)로 등록하며 시맨틱 버저닝(semantic versioning)을 따른다.

---

## 5. Implementing Contract-Driven Tooling

> 기계 판독 가능한 컨트랙트의 메타데이터로 데이터 형태와 무관하게 작동하는 범용 익명화 서비스를 구현한다.

- 익명화 함수는 컨트랙트를 직접 읽어 anonymization_strategy(email, hex)에 따라 필드를 익명화한다.
- 같은 코드가 어떤 컨트랙트에도 작동하며, name은 hex 인코딩, email은 anonymized+id 형태로 치환된다.
- 동일 방식으로 데이터 품질 체크, 접근 제어 자동화, SLA(Service-Level Agreement) 수집·보고, 데이터 전송, 백업 등을 구축할 수 있다.
- 이런 툴링 제공으로 생성자는 공통 작업 구현 대신 품질 데이터 생성에 집중할 수 있다.

```
def anonymize(event, data_contract):
  for name, metadata in data_contract.fields().items():
    if metadata.get('anonymization_strategy') == 'hex':
      anonymized[name] = event[name].encode("utf-8").hex()
```

---

## Summary (핵심 정리)

- 커스텀 YAML 컨트랙트 하나로 BigQuery 테이블, 검증 라이브러리, 스키마 레지스트리, 익명화 서비스를 모두 구동했다.
- Pulumi(IaC), JSON Schema, Confluent Schema Registry로 리소스 생성과 스키마 진화·호환성 관리를 실습했다.
- 어떤 데이터 툴링·통합도 같은 방식으로 구축 가능하며, 다음 장에서는 조직에서의 실제 도입을 다룬다.
