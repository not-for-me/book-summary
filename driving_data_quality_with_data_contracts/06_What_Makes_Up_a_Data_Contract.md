# 06. What Makes Up a Data Contract

## 챕터 개요 (3줄 요약)

- 데이터 컨트랙트의 큰 부분은 데이터 구조를 정의·문서화하는 스키마(schema)이며, 여러 오픈소스 포맷으로 정의할 수 있다.
- 데이터는 진화하므로 버전 관리로 스키마 진화(evolution)를 다루고, 소비자가 신뢰할 수 있도록 마이그레이션을 관리한다.
- 컨트랙트는 스키마를 넘어 거버넌스·통제 메타데이터를 담으며, 기계 판독 가능해 다양한 툴·서비스와 통합된다.

---

## 1. The Schema of a Data Contract

> 스키마는 데이터의 구조(필드와 타입)를 정의하며, 모든 데이터 컨트랙트는 스키마를 가져야 한다.

### Defining a schema

- 스키마는 최소한 가용 필드의 전체 목록과 데이터 타입을 담아 데이터 품질의 최소 기준을 보장한다.
- Protocol Buffers(protobuf), Apache Avro는 직렬화 프레임워크로 컴팩트한 바이너리 인코딩을 제공한다.
- Apache Avro는 필드에 doc(문서)을 추가해 소비자가 맥락을 이해하도록 돕는다.
- JSON Schema는 JSON 직렬화에 더해 데이터 검증(validation: pattern, enum, required 등) 기능을 제공한다.
- 검증은 라이브러리가 원천 시스템에서 잘못된 데이터 방출을 막고 생성자에게 즉시 알리는 데 쓰인다.
- 고급 메타데이터 요구로 인해 대부분의 구현은 YAML, Jsonnet, Python/TypeScript 같은 상위 정의 언어를 쓴다.

### Using a schema registry as the source of truth

- 스키마는 소스 코드 생성, 직렬화/역직렬화, CI(Continuous Integration) 검증, 리소스 정의 등에 활용된다.
- 여러 애플리케이션이 같은 버전을 참조하도록 중앙 스키마 레지스트리(schema registry)를 단일 진실 공급원으로 둔다.
- 레지스트리는 신규/갱신 스키마 발행, 특정/최신 버전 조회 기능을 갖춰야 한다.
- 예시: Confluent Schema Registry, Iglu(Snowplow), AWS Glue Schema Registry.
- 단점: 사용 증가 시 성능 병목·단일 장애점(single source of failure)이 될 수 있어 완화 설계가 필요하다.

```
message Customer {
  string id = 1;
  string name = 2;
  string email = 3;
  string language = 4;
}
```

---

## 2. Evolving Your Data Over Time

> 버전 관리로 스키마 진화를 추적하며, 파괴 변경에는 의도적 마찰을 두어 소비자에게 안정성을 제공한다.

### Evolving your schemas

- 비파괴 변경(non-breaking change): 신규/이전 버전 데이터가 서로 손실 없이 읽힌다(예: 선택적 필드 추가, 기본값 있는 비필수 필드 제거).
- 비파괴 변경은 영향이 작아 낮은 마찰로 적용 가능하다.
- 파괴 변경(breaking change): 기존 소비자에 영향(예: 필수 필드 제거, 데이터 타입 변경)을 주어 파이프라인·대시보드를 깨뜨릴 수 있다.
- Apache Avro·Protocol Buffers는 명세에 스키마 호환성 규칙을 명확히 정의한다.

### Migrating your consumers

- 생성자는 새 버전이 필요할 때 먼저 소비자와 변경을 논의해 요구를 충족하는지 확인한다.
- 마이그레이션 계획은 변경 크기·데이터 중요도·소비자 수에 따라 달라진다(병행 운영 기간, 마이그레이션 라이브러리 제공 등).
- 핵심은 생성자와 소비자가 합의한 계획이 존재하는 것이며, 이는 두 역할을 가깝게 만드는 목표와 일치한다.
- 이 마찰은 의도적이며, 민첩성보다 안정성을 우선해 신뢰할 수 있는 데이터를 제공한다.

```
v1 ----(non-breaking: add field)----> v1 readers unaffected
v1 ----(breaking: remove field)-----> migration plan + dual-write period needed
```

---

## 3. Defining the Governance and Controls

> 모든 컨트랙트는 소유자(owner)를 가지며, 거버넌스·통제 메타데이터를 담아 기계 판독 가능한 단일 진실 공급원이 된다.

- 컨트랙트에 담을 수 있는 메타데이터: 버전, SLA(Service-Level Agreement), 접근 방식, 기본키, 관련 엔티티, 시맨틱, 개인 데이터 여부, 분류, 보존 기간, 삭제·익명화 정책, 물리적 위치.
- 직렬화용 스키마 포맷은 이런 광범위한 메타데이터를 담도록 설계되지 않아 YAML, Jsonnet 등 상위 언어를 사용한다.
- 언어 선택은 조직의 기존 도구에 따른다(GoCardless는 인프라 플랫폼 언어인 Jsonnet 사용; YAML도 좋은 선택).
- 커스텀 스키마는 생성자를 위한 정의 인터페이스이자 소비자의 발견·이해용이며, 기계 판독 가능해 툴 연동이 된다.
- 단일 컨트랙트를 Protocol Buffers, BigQuery 테이블, JSON Schema 등 어떤 포맷으로도 변환해 오픈 생태계를 활용한다.
- 컨트랙트는 단일 진실 공급원으로 남고 나머지는 모두 파생되어, 데이터 핸들링·삭제·익명화 툴링을 구동한다.

```
                 +--> Protocol Buffers (Pub/Sub schema validation)
[Data Contract] -+--> BigQuery table schema
 (source of truth)+--> JSON Schema (in-code validation)
                 +--> internal data-handling tooling
```

---

## Summary (핵심 정리)

- 데이터 컨트랙트의 핵심은 스키마이며, 오픈소스 포맷과 스키마 레지스트리(단일 진실 공급원)로 정의·관리한다.
- 데이터 진화는 버전 관리로 다루되, 파괴 변경에는 마찰과 마이그레이션 계획을 두어 소비자 안정성을 지킨다.
- 컨트랙트는 거버넌스·통제 메타데이터를 담아 기계 판독 가능하며, 어떤 포맷으로든 변환해 툴·서비스와 통합된다.
