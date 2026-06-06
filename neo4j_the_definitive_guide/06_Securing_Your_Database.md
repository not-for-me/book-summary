# 06. Securing Your Database

## 챕터 개요 (3줄 요약)

- Neo4j 보안을 STRIDE 위협 모델(Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege)에 따라 체계적으로 다룬다.
- 인증(authentication)·인가(RBAC)·통신 암호화(SSL/TLS)·저장 데이터 암호화·Cypher 주입 방어·감사 로그 등 방어 메커니즘을 설명한다.
- 최소 권한(least privilege), 불변 권한(immutable privilege), 세분화된 접근 제어(fine-grained access control), 정기 권한 검토, 패치 적용 같은 운영 모범 사례로 마무리한다.

---

## 1. Spoofing — Authentication

> 스푸핑은 공격자가 다른 사용자로 위장하는 위협으로, 강력한 인증으로 방어한다.

- 인증 제공자(auth provider)로 네이티브(native) 방식 또는 LDAP 제공자를 사용할 수 있다.
- Neo4j 브라우저를 통한 접근도 보안 설정으로 보호해야 한다.
- 인증(authentication, 사용자 신원 확인)은 인가에 앞선 첫 번째 방어선이다.
- 모범 사례에는 강한 자격 증명 관리와 접근 경로 최소화가 포함된다.

---

## 2. Tampering — Integrity Protection

> 탬퍼링은 데이터나 쿼리를 무단·악의적으로 변경하는 위협이다.

- 통신 채널 보안(securing communication channels): 클라이언트/관리 도구와의 전송 중 데이터에 SSL/TLS를 적용한다. 기본 포트(bolt 7687, https 7473, cluster 5000/6000/7000/7688, backups 6362)는 공격 표면 축소를 위해 변경하는 것이 좋다.
- 저장 데이터 보안(data at rest): 디스크에 저장된 데이터를 암호화해 보호한다.
- 일관성 검사(consistency checks)로 데이터 무결성을 검증한다.
- Cypher 주입(injection) 방어: 사용자 입력을 문자열로 연결하지 말고 쿼리 파라미터를 사용한다.
- 역할 기반 접근 제어(RBAC, Role-Based Access Control): 권한(privilege)의 집합인 역할(role)을 사용자에게 부여해 수행 가능한 작업을 통제한다. 서비스 계정, 쓰기 권한, `LOAD CSV` 권한, 제약(constraint), 백업도 함께 관리한다.

---

## 3. Repudiation — Accountability

> 부인(repudiation)은 사용자가 수행한 행위를 부정하고 이를 증명할 수 없는 위협이며, 부인 방지(non-repudiation)로 대응한다.

- 모든 로그를 구성해 수행된 모든 행위의 추적 흔적(audit trail)을 남긴다.
- 감사 로그(audit logs)는 책임 추적성, 추적성, 법적 컴플라이언스 유지에 필수적이다.
- 로깅을 통해 누가 무엇을 했는지 증거를 확보한다.

---

## 4. Information Disclosure — Confidentiality

> 정보 노출은 민감 정보가 비인가 당사자에게 드러나는 위협이다.

- 쿼리 로그(query logs)는 접근 권한자에게 정보를 노출할 수 있으므로 주의해야 한다.
- 세분화된 접근 제어(fine-grained access control): 노드/관계의 민감한 속성값에 적절한 읽기/쓰기 권한을 부여해 경로 순회로 우발적으로 노출되지 않게 한다.
- 속성 암호화(property encryption): 애플리케이션 계층에서 강한 알고리즘으로 민감 속성을 암호화 후 저장하고 읽을 때 복호화한다 — 단, 암호화된 속성은 검색이 불가하다.

---

## 5. Denial of Service & Elevation of Privilege

> DoS는 시스템을 정당한 사용자가 못 쓰게 만드는 위협이고, 권한 상승은 의도보다 높은 권한을 획득하는 위협이다.

- 서비스 거부(DoS, Denial of Service): 자원 고갈로 크래시를 유발하므로 트랜잭션 타임아웃(`db.transaction.timeout`) 등으로 완화한다.
- 권한 상승(elevation of privilege): 불변 권한(immutable privileges)으로 권한 관리자의 행위를 제한한다.
- 최소 권한(least privileges): Neo4j는 `GRANT`가 있고 명시적 `DENY`가 없을 때 접근을 허용하므로, 명시적 `DENY`로 광범위 접근을 차단한다.
- 확장(extensions), 사용자/권한 정기 검토, 파일 권한, 패치 적용으로 공격 표면을 관리한다.

```
STRIDE threat model for Neo4j:
  Spoofing            -> Authentication (native / LDAP)
  Tampering           -> SSL/TLS, encryption at rest, RBAC, anti-injection
  Repudiation         -> Audit logs, query logs
  Information Disclos. -> Fine-grained access, property encryption
  Denial of Service   -> Transaction timeouts, resource limits
  Elevation of Priv.  -> Least privilege, immutable privileges, DENY
```

---

## Summary (핵심 정리)

- STRIDE 위협 모델을 기준으로 인증·인가(RBAC)·전송/저장 암호화·주입 방어·로깅을 계층적으로 적용해야 한다.
- 민감 속성은 세분화된 접근 제어와 애플리케이션 계층 암호화로 보호하되, 암호화 속성은 검색 불가라는 트레이드오프가 있다.
- 최소 권한과 명시적 `DENY`, 불변 권한, 정기 권한 검토와 패치로 권한 상승·DoS 위협을 지속적으로 관리한다.
