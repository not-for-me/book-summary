# 07. A Contract-Driven Data Architecture

## 챕터 개요 (3줄 요약)

- 컨트랙트 주도 데이터 아키텍처(contract-driven data architecture)는 컨트랙트의 메타데이터로 데이터 플랫폼을 구동하는 패러다임 전환이다.
- 포인트 솔루션(point solution)의 중복 대신, 메타데이터 기반의 범용 툴링을 데이터 인프라팀이 제공한다.
- 자동화(automation), 가이드라인·가드레일, 일관성(consistency)이라는 세 원칙과 셀프서브 인프라로 생성자의 자율성을 실현한다.

---

## 1. A Step-Change in Building Data Platforms

> 데이터별 포인트 솔루션을 반복 구축하면 데이터 엔지니어링팀이 병목이 되므로, 컨트랙트 메타데이터로 구동되는 범용 툴링이 필요하다.

### Building generic data tooling

- 대부분의 파이프라인·관리 서비스는 특정 데이터에 종속된 포인트 솔루션으로 만들어져 중복이 발생한다.
- 이로 인해 데이터 엔지니어링팀만이 데이터를 적재·관리할 수 있어 병목이 되고 접근성이 제한된다.
- 컨트랙트에 warehouse_path, 백업 일정·만료, 익명화 전략 등 메타데이터를 담으면 임의 형태의 데이터를 처리하는 범용 서비스를 만들 수 있다.
- 예: 적재 서비스는 스키마로 테이블을 동기화하고, 백업 서비스는 cron으로 실행하며, 핸들링 서비스는 익명화 전략을 적용한다.

### Introducing a data infrastructure team

- 생성자의 자율성을 위한 셀프서브 툴링은 프로덕트 마인드셋으로 구축·제공해야 한다.
- 전담 데이터 인프라팀(소프트웨어 엔지니어, SRE(Site Reliability Engineer))을 두어 다른 인프라 플랫폼팀과 함께 일한다.
- GoCardless에서는 이 팀이 데이터 엔지니어가 아닌 소프트웨어 엔지니어로 구성되어 제품 개발 조직에 속한다.

```
warehouse_path: sales_data.salesforce.customers
backups: { schedule: @daily, expire: P60D }
fields: { email: { personal_data: true, anonymization_strategy: email } }
```

---

## 2. Case Study & Promoting Autonomy Through Decentralization

> GoCardless의 Data Platform Gateway는 자율성을 노렸지만 또 다른 병목이 되었고, 그 교훈이 데이터 컨트랙트 설계로 이어졌다.

- Data Platform Gateway(2018): HTTP API로 데이터를 웨어하우스에 푸시하고 스키마 레지스트리로 검증했다.
- 초기에는 성공했으나 스키마 생성이 거의 0으로 줄고, Pub/Sub 직접 소비도 없었다.
- 실패 원인 ① 설계 선택 이견(Apache Avro의 Ruby 지원 부족, 한 단계 깊이 스키마 제약).
- 실패 원인 ② 리소스·데이터 소유권 부재(중앙팀을 거쳐야 변경 가능).
- 실패 원인 ③ SLO(Service-Level Objective)·기대치 미정의, 공유 리소스로 인한 광범위 영향.
- 결국 데이터 인프라팀이 새 병목이 되었고, 생성자-소비자 간 거리는 여전했다.
- 다음 반복의 네 방향: 탈중앙화(decentralization), 소유권(ownership), 유연성(flexibility), 격리(isolation).
- 핵심 통찰: 필요한 것은 스키마의 '모양'이 아니라 기계 판독 가능한 메타데이터로 정의된 '맥락'이었다.

```
Before: [Central Data Infra Team] = new bottleneck (gateway)
After:  decentralized contracts -> resources in each team's own infra (isolated)
```

---

## 3. Principles of a Contract-Driven Data Architecture

> 컨트랙트 주도 아키텍처의 이점은 자동화, 가이드라인·가드레일, 일관성이라는 세 원칙으로 달성된다.

### Automation

- 웨어하우스 테이블, Kafka/Pub/Sub 토픽, SLO 메트릭, 접근 제어를 컨트랙트를 단일 진실 공급원으로 자동 생성·동기화한다.
- 백업, 데이터 이동, 보존 기간 경과 시 익명화·삭제 등 정기 작업을 소규모 서비스로 자동화한다.

### Guidelines and guardrails

- 모두가 전문가일 수 없으므로 툴링이 올바른 방향으로 안내(가이드라인)하고 보호(가드레일)한다.
- CI(Continuous Integration) 체크로 개인 데이터 분류·익명화 전략 정의 여부 등을 검증한다.
- 중앙팀 리뷰 없이도 빠르게 움직이며 리스크를 크게 줄인다.

### Consistency

- 표준 툴링으로 모든 소비자가 데이터 발견·기대치 조회·소유자 확인·권한 요청 방식을 일관되게 안다.
- 생성자도 데이터셋 간 전환 시 맥락 손실 없이 동일한 도구를 사용한다(golden path).
- 일관성은 포인트 솔루션을 줄여 투자 대비 효과(ROI)와 데이터 인프라팀의 가치를 높인다.

---

## 4. Providing Self-Served Data Infrastructure

> 셀프서브가 자율성의 핵심이며, GoCardless는 Jsonnet 컨트랙트로 리소스를 중앙 리뷰 없이 프로비저닝한다.

- 가드레일이 리스크를 관리하므로 중앙팀 리뷰 없이 생성자를 신뢰하고 주인의식을 부여할 수 있다.
- 생성자는 컨트랙트에 `+ withPubSub() + withBigQuery()`를 추가해 리소스를 셀프서브로 프로비저닝한다.
- 데이터 핸들링·백업 서비스는 합리적 기본값으로 자동 생성되며 컨트랙트에서 설정 변경(예: @weekly 백업)이 가능하다.
- 컨트랙트는 데이터 카탈로그·옵저버빌리티 플랫폼 등 중앙 서비스와 통합된다.
- 컨트랙트를 Git에 머지하면 중앙 리뷰 없이 리소스가 생성된다.

```
schema { versions: [ new_version('1', ...) + withPubSub() + withBigQuery() ] }
```

---

## Summary (핵심 정리)

- 컨트랙트 주도 아키텍처는 컨트랙트 메타데이터로 범용 툴링을 구동하는 데이터 플랫폼 구축의 패러다임 전환이다.
- 실패한 Data Platform Gateway 사례에서 자율성의 중요성을 배워, 자동화·가드레일·일관성 세 원칙을 채택했다.
- 셀프서브 인프라가 자율성의 핵심이며, 다음 장에서는 이 패턴의 샘플 구현을 다룬다.
