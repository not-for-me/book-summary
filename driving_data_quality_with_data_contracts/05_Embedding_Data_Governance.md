# 05. Embedding Data Governance

## 챕터 개요 (3줄 요약)

- 데이터 거버넌스는 데이터를 접근 가능·정확·안전·규정 준수 상태로 유지하는 사람·프로세스·표준·기술의 조합이다.
- 데이터 컨트랙트에 거버넌스 메타데이터를 내장하면 접근 제어·삭제·익명화 등 다양한 자동화가 가능해진다.
- 거버넌스 책임을 생성자에게 두고 데이터 거버넌스 위원회(council)가 지원하는 연합형 거버넌스(federated governance)로 민첩성과 리스크 관리의 균형을 맞춘다.

---

## 1. Why We Need Data Governance

> 데이터 거버넌스는 데이터의 보안·규정 준수를 보장하는 핵심 활동이지만, 전통적 방식은 종종 병목이 되어 제대로 달성되지 못한다.

### The requirements of data governance

- 데이터 거버넌스는 데이터를 접근·사용·정확·일관·보안·규정 준수 가능하게 지원한다.
- 법적으로 개인 데이터(personal data, 광의)와 비개인 데이터(대부분 비규제)로 나뉜다.
- 개인 데이터(personal data)와 PII(Personally Identifiable Information)는 법적으로 다르며, PII는 개인을 식별 가능한 부분집합이다.
- GDPR(General Data Protection Regulation)은 광의의 개인 데이터를 규제한다.
- 추적 의무: 데이터 취득 경로, 처리 목적, 변환(익명화/가명화) 여부, 보존 기간, 물리적 위치, 리스크 평가 등.
- EU AI(Artificial Intelligence) Act 등 규제가 강화되고 있어 유연한 거버넌스 구조를 미리 갖춰야 한다.

### How governance is typically applied

- 보통 중앙 팀이 수작업으로 인벤토리/카탈로그를 관리해 정확·완전한 목록을 갖춘 조직이 드물다.
- 거버넌스가 '게이트키퍼(gatekeeper)' 위원회로 작동해 리스크 축소만 추구하며 데이터 사용을 늦추는 나쁜 평판이 있다.
- 결과는 둘 중 하나: 병목이 되어 사용 가능 데이터가 줄거나, 일부 핵심 데이터셋에만 거버넌스를 적용해 나머지가 비준수 상태가 된다.

---

## 2. Promoting Data Governance Through Data Contracts

> 컨트랙트에 거버넌스 메타데이터를 정의해 두면 정확·최신 상태로 유지되고, 이를 기반으로 자동화 툴링을 구축할 수 있다.

- 컨트랙트에 담는 메타데이터: 개인 데이터 여부, 관련 엔티티, 접근 권한과 만료, 분류(기밀/비밀/공개), 보존 기간, 삭제·익명화 정책, 물리적 위치.
- 소유자가 메타데이터를 채우게 해 정확·완전·진실한 데이터 자산 인벤토리를 구축한다.
- 메타데이터는 기계 판독 가능(machine-readable)해 검증·가드레일·툴링 자동화를 가능케 한다.
- 예시: 분류에 따라 접근 권한을 자동 부여·동기화하고, 보존 기간 경과나 삭제 요청 시 삭제·익명화를 자동 실행한다.
- 메타데이터는 단일 진실 공급원(source of truth)으로서 중앙 프라이버시 툴에 모여 내부·외부 감사에 활용된다.

```
fields:
  name:
    personal_data: true
    anonymization_strategy: hex
  email:
    personal_data: true
    anonymization_strategy: email
```

---

## 3. Assigning Responsibility for Data Governance

> 거버넌스 책임을 데이터를 가장 잘 아는 생성자에게 두고, 중앙 위원회가 표준·툴링으로 지원하는 탈중앙 모델을 채택한다.

### Responsibilities of the data generators

- 중앙 팀은 병목이 되고 맥락이 부족하므로, 생성자가 거버넌스 책임을 진다.
- 생성자가 할 일은 메타데이터(개인 데이터 여부, 민감도 분류, 삭제·익명화 정책)를 채우고 최신화하는 것뿐이며 나머지는 자동화된다.

### The data governance council

- 데이터 거버넌스 위원회는 정책·표준을 정의하고 툴링 도입 영역을 식별하는 교차기능(cross-functional) 그룹이다.
- 구성: 데이터 생성·소비 영역 대표(데이터 제품 매니저), 법무·프라이버시·보안 전문가, 데이터 플랫폼팀 PM/테크리드.
- 설립 단계: 범위·목표 명확화 → 균형 잡힌 멤버 선정(최대 10명) → 임원 스폰서십 확보 → 투명하게 운영·공유.

### Federated data governance

- 생성자에게 로컬 분류 결정 자율성을 주고, 중앙 위원회가 표준·문서·가드레일로 지원한다.
- 생성자는 전문가가 아니라 실수할 수 있으나, 이는 속도와 리스크 관리 사이의 좋은 트레이드오프다.
- 이것이 데이터 메시(data mesh)의 원칙 중 하나인 연합형 거버넌스(federated data governance)다.

```
[Central Governance Council] --policies/standards/tooling--> [Data Generators (local decisions)]
        (defines)                                              (classify & manage own data)
```

---

## Summary (핵심 정리)

- 데이터 컨트랙트에 거버넌스를 내장하면 분류·메타데이터가 정확·최신으로 유지되고 관리 자동화를 구동할 수 있다.
- 거버넌스 책임은 맥락을 가진 생성자에게 부여하되, 중앙 위원회가 정책·표준·기술로 지원한다.
- 이로써 로컬 의사결정과 중앙 지원이 결합된 연합형 거버넌스로 민첩성과 리스크 관리의 균형을 달성한다.
