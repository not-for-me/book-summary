# 10. To Production

## 챕터 개요 (3줄 요약)
- 프로덕션은 실제 사용자가 코드와 상호작용하는 궁극적 시험대이며, 개발 환경과의 차이를 메우는 것이 핵심이다.
- 성능 최적화·환경별 설정·오류 처리/로깅·보안을 개발 초기부터 고려해 프로덕션 준비 코드를 만든다.
- 배포 파이프라인(환경, 버전 관리, 자동화, CI/CD), 배포 전략, 모니터링·유지보수로 안정적 배포를 달성한다.

---

## 1. The Complexities of Production Environments
> 소프트웨어의 진정한 가치는 실제 사용자가 있는 프로덕션에서의 성능으로 결정된다.

### Users Are Unpredictable
- 어떤 테스트나 시뮬레이션도 실제 사용의 예측 불가능성을 재현할 수 없어 사용자가 궁극적 시험이다.
- 요인: 실세계 데이터, 규모/동시성, 다양한 환경, 환경 드리프트(environmental drift), 예상 밖 사용 사례.
- 카나리 릴리스, A/B 테스트, 견고한 관찰성(observability)으로 위험을 최소화한다.

### "But It Works on My Machine"
- "내 컴퓨터에선 됨"은 사용자 문제를 해결하지 않으며 환경 간 불일치 원인을 찾아야 한다.
- 변수: 환경 차이(OS/리소스/DB 버전), 설정 관리, 동시성/부하, 데이터 다양성, 외부 의존성, 보안 제약.
- 해결: 컨테이너화(Docker), 관찰성, 환경 패리티(스테이징), CI/CD.

---

## 2. Building Production-Ready Code
> 부모가 아이를 학교 보낼 준비를 하듯, 코드를 프로덕션의 예측 불가능한 도전에 대비시켜야 한다.

### Performance Optimization
- 성능 최적화는 나중에 켜는 다이얼이 아니라 코드 작성 전부터 가지는 사고방식이다.
- 전략: 비동기 프로그래밍, 네트워크 호출 감소(배칭), 캐싱, 쿼리 최적화, 코드 압축(minification)/번들링, 지연 로딩, CDN(Content Delivery Network).

### Environment-Specific Configurations
- "Configuration is code"이지만 환경마다 달라지므로 자격증명을 하드코딩하지 않는다.
- 설정 파일(환경별 application.yaml), 환경 변수(${VAR})로 민감 정보를 분리한다.
- 기능 플래그(feature flags): 코드 배포 없이 런타임에 기능 활성화 — 릴리스/실험/권한 플래그.
- 시크릿(secrets)은 절대 코드/버전 관리에 저장하지 말고 AWS Secrets Manager, HashiCorp Vault 등을 사용한다.

### Error Handling and Logging
- 파레토 법칙: 해피 패스는 20%이지만 엣지 케이스/오류 처리가 노력의 80%를 차지한다.
- 우아한 성능 저하(graceful degradation), 의미 있는 오류 메시지, 디버깅용 로깅, 폴백 옵션.
- 로깅 레벨: ERROR, WARN, INFO, DEBUG — 민감 정보, PII(Personally Identifiable Information), 대형 객체는 로깅하지 않는다.

### Security Essentials
- 보안은 처음부터 개발 프로세스에 내장하며 OWASP(Open Web Application Security Project)가 좋은 자료다.
- HTTPS는 TLS(Transport Layer Security)로 암호화하며(SSL은 구식, 사용 금지), 어디서나 HTTPS를 쓴다.
- 자체 보안 구현 금지 — Spring Security 등 검증된 솔루션 사용, BCrypt로 비밀번호 인코딩.
- 강력한 비밀번호 관리, 계정/세션 보안, MFA(Multi-Factor Authentication)로 다층 방어.
- PII 마킹, AES(Advanced Encryption Standard) 암호화, 규정 준수(GDPR, CCPA, HIPAA, PCI DSS), SBOM(Software Bill of Materials).

---

## 3. Deployment Pipeline
> 배포 파이프라인은 코드를 안전하고 신뢰성 있게 프로덕션으로 옮기는 로드맵이다.

### Deployment Environments
- 로컬 개발 → 테스팅/QA → 스테이징(드레스 리허설) → 프로덕션(메인 스테이지).
- 환경 간 이동의 핵심은 각 단계에 맞는 올바른 설정을 보장하는 것이다.

### Version Control Strategies
- Git Flow(다중 장기 브랜치), GitHub Flow(단순), 트렁크 기반 개발, 릴리스 트레인.
- Git Flow 코어 브랜치: main(프로덕션 준비, 진실의 원천), develop(일상 작업 공간).
- 지원 브랜치: feature(개발 대부분), release(안정화/준비), hotfix(긴급 수정, main에서 분기).
- 브랜치 이름은 명확하게(feature/user-authentication > my-branch/stuff).

```
Git Flow:
  feature/* --> develop --> release/* --> main (production)
                              hotfix/* --> main + develop
```

### Deployment Automation
- 수동 배포는 시간 소모적·오류 유발·스트레스 — 배포 스크립트로 반복 가능한 공장처럼 자동화한다.
- 롤백(rollback) 절차: 최근 백업을 찾아 중지→복원→시작→검증하며, 정기적으로 테스트한다.

### Deployment Strategies
- All-at-once(big bang): 전체를 한 번에 교체 — 단순하지만 고위험, 소규모/다운타임 허용 시.
- Gradual(phased): 일부 사용자/서버에 먼저 롤아웃 — 카나리/롤링 배포, 대규모/무중단에 적합.
- Blue-green: 동일 환경 둘을 유지하고 전환 — 미션 크리티컬 앱에 매끄러운 경험.

### CI/CD
- CI(Continuous Integration)는 팀 변경을 자동 통합·테스트, CD(Continuous Deployment/Delivery)는 릴리스를 준비/배포.
- 이점: 일관성, 자동 테스트, 스트레스 감소, 통합 이슈 조기 발견, 코드→프로덕션 시간 단축.
- 기본 워크플로: Code → Build → Test → Package → Deploy (예: GitHub Actions).
- 고급 패턴: 카나리 릴리스, 블루-그린 배포 — 단순하게 시작해 점진적으로 발전시킨다.

---

## 4. Production System Monitoring and Maintenance
> 프로덕션 배포는 끝이 아니라 시작이며, 모니터링과 유지보수는 지속적 여정이다.

### Monitoring
- 사용자는 문제를 알려주지 않고 그냥 떠나므로 모니터링/로깅이 필수다.
- 실시간 모니터링(현재 상태)과 로그(과거 기록)를 함께 사용한다 — 개별 메트릭만으론 전체 그림을 못 본다.
- 규칙: 필요한 정보 로깅, 타임스탬프/사용자 식별자 포함, 오류는 오류로 표시, 민감 정보 제외, 오래된 로그 삭제.

### System Maintenance
- OS 업데이트와 보안 패치가 첫 방어선이며, 보안 패치는 즉시 적용한다.
- 의존성 관리는 가장 간과되는 영역 — SBOM으로 추적하고 취약 버전(예: Spring Boot 2.5.5)을 업데이트한다.
- Maven/Gradle 의존성 분석기와 자동 스캔(GitHub Actions)으로 취약점을 조기에 잡는다.
- 업데이트 정책 수립: 정기 간격, 긴급 패치 절차, 자동 스캔, 문서화 — 예방이 복구보다 저렴하다.

---

## Summary (핵심 정리)
- 코드를 프로덕션으로 옮기는 일은 처음엔 위협적이지만 준비가 좋은 배포 프로세스의 핵심이다.
- 첫날부터 프로덕션 준비를 생각하고(성능·보안·오류 처리), 환경별 설정·모니터링·버전 관리·CI/CD 자동화를 적용한다.
- 사용자는 "내 컴퓨터에서 됨"에 관심이 없고 자기 환경에서 작동하는지를 신경 쓰며, 각 실패는 더 나은 엔지니어가 되는 기회다.