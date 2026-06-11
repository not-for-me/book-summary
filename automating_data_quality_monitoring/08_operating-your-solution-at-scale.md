# 08. Operating Your Solution at Scale

## 챕터 개요 (3줄 요약)
- 운영은 acquisition(build vs buy)→configuration→enablement→장기 개선의 단계를 따르며, 각 단계의 결정이 비용·성능·신뢰를 좌우한다.
- build vs buy, vendor deployment model(SaaS/in-VPC/hybrid)은 control·security·data privacy의 trade-off다.
- 중요 table 식별, 최신 data만 모니터링, API/UI 이중 config, 체계적 onboarding, initiative·metric·scorecard로 data quality를 지속 개선한다.

---

## 1. Build Versus Buy
> 문제와 solution space를 충분히 이해한 뒤 자체 구축할지 구매할지 결정한다.

- build는 대개 scratch가 아니라 open source(Great Expectations, Deequ) 기반. 장점: roadmap 완전 통제, ML model·UX·내부 system 호환·비용·custom 기능을 원하는 대로, 불필요한 것에 비용 안 듦.
- 단점: core competency에서 시간 이탈(유지보수 인력 확보·유지 어려움), 팀별 제각각 구축으로 통합·투명성 저하, 24/7 지원·교육 부담, 끊임없는 혁신 필요, vendor만큼 성능·비용 최적화 어려움, 결국 action 안 되는 email만 양산하고 UX/visualization 복잡성에서 좌초.
- vendor ecosystem 성장으로 buy가 점점 매력적 — 총 운영비 더 낮고 1년+ 기다림 없이 즉시 시작. cloud warehouse도 기능 추가 중이나 data quality 전문이 아니라 혁신·UX·edge case에 불리. **lock-in** 우려(다중 warehouse, 향후 migration 시 전환 비용).

### 1.1 Vendor Deployment Models
> security·data privacy에 직결되는 vendor deployment model을 주의 깊게 봐야 한다.

- **SaaS**: query·분석·presentation 모두 vendor 환경. 가장 단순하나 vendor가 data·metadata에 read access — table/column 이름이 경쟁 정보일 수 있어 대기업엔 대개 nonstarter.
- **Fully in-VPC/on-prem**: 전체 application이 고객 통제 환경에서 실행. Docker/Kubernetes로 관리·auto-upgrade 용이. 보안 자세 조절(remote access 허용/차단, no egress — telemetry조차 차단 시 고객 부담·비용 증가). 대부분 보안 민감 기업이 선택, 관리 복잡성은 learning curve. **data 노출 안 됨을 보장하는 유일한 방법.**
- **Hybrid**: query·분석은 고객 환경(in-VPC), presentation은 vendor cloud(SaaS). 그러나 summary statistic·sample·metadata가 여전히 환경을 떠나야 해 SaaS와 크게 다르지 않음 — UX를 제약하거나 민감 data 노출 둘 중 하나로 타협.

---

## 2. Configuration
> 대용량 환경에선 어떤 table·어떤 data를 모니터링할지가 비용·성능을 크게 좌우한다.

### 2.1 Determining Which Tables Are Most Important
> 전체는 metadata observability로, deep monitoring은 중요 table만 설정하는 것이 비용 효율적이다.

- 중요 table부터 시작하면 onboarding·신뢰 구축에도 유리. 소규모는 자명, 대규모는 지식이 분산 — data **consumer**가 가장 관심 많으므로 SME가 직접 monitoring 설정하게. **SQL query log의 "heat"**(자주 query되는 table/column/segment)가 중요도의 좋은 proxy.

### 2.2 Deciding What Data in a Table to Monitor
> 대용량 table은 최신 data(보통 전날)만 모니터링해 warehouse 부하·비용을 줄인다.

- 대부분 issue는 신규 도착 data에서 발생하므로 최신만으로도 충분(예외: updated-in-place는 별도 전략). 최신 data 식별엔 time partitioning 필요(partitioning scheme이 복잡할 수 있음). time column 없는 table은 매 갱신 시 전체 check + snapshot 비교.

### 2.3 Configuration at Scale
> 10,000 table을 UI로 일일이 설정할 수 없으므로 programmatic config가 필요하다.

- robust API(CLI·YAML)로 table 설정·custom check 추가/삭제/갱신, 기존 dbt rule migration. config를 Git으로 버전 관리 → audit·code-review. 단 비개발 business user를 위해 **code config + UI self-service** 둘 다 제공.

---

## 3. Enablement
> 다양한 stakeholder·여러 팀에 플랫폼을 도입하며, 사용자가 많아질수록 data quality 투자가 커진다.

### 3.1 User Roles and Permissions
> 역할·권한(admin~viewer)을 초기에 정의한다.

- admin 지정·백업, 팀별 admin 여부, 신규 user 기본 role, 접근 escalation 절차 결정. 필요한 것만 접근 허용(예: HR 등 민감 table siloing).

### 3.2 Onboarding, Training, and Support
> top-down 전사 확산 또는 small start 후 viral adoption — 문화·긴급도·자원·팀 연결성에 따라 선택한다.

- 일반 전략: **initial kickoff**(이해관계자 buy-in·계획·demo), **live training**(팀별 매핑, config 담당자 초대), **on-demand curriculum**(글·영상, self-paced), **office hours**(주간, 녹화 공유), **ongoing support**(전용 Slack, roadmap 리뷰, error/성능 모니터링·SLA 예: P1 1시간). 궁극적으로 engagement 향상·silent failure 방지.

---

## 4. Improving Data Quality Over Time
> 좋은 플랫폼은 시작일 뿐, 지속 개선과 data quality 문화가 핵심이다.

### 4.1 Initiatives
> 단일 도구를 넘어 일하는 방식을 바꾸는 data health initiative에 commit한다.

- third-party source audit, data catalog 등 인프라 투자, debugging runbook 문서화(silo 지식 방지), 각 table owner·triage 절차 지정, producer-consumer 간 data contract. data quality 투자가 운동→식단처럼 연쇄 변화의 momentum을 만듦.

### 4.2 Metrics
> 진행 상황 측정은 타 팀 독려·리더십 영향 입증에 중요하다.

- **Triage and resolution**: 탐지 수보다 issue의 **magnitude와 해결 속도**가 핵심 성공 지표. 명확한 절차·response SLA를 전사 공유.
- **Executive dashboards**: coverage, WoW issue 추세, issue 많은 table 등 고수준 지표.
- **Scorecards**: table별 health를 단일 값/등급으로. check pass/fail에 priority 가중치 부여, 카테고리별 분류(Fig 8-1)도 가능하나 분류 결정이 까다로움. one-size-fits-all 드묾 → 최소한 API/export로 check 이력·priority·tag·pass/fail 제공해 custom governance dashboard 구성.

---

## 5. From Chaos to Clarity
> 운영 과제 해결 도구를 갖췄으니 조직의 data quality 이야기를 바꿀 준비가 됐다.

- 책 전체 요약: Ch1(왜 중요), Ch2(rule을 넘어 ML 포함 종합 solution), Ch3(ROI 평가), Ch4(ML 알고리즘), Ch5(현실 data 튜닝·테스트), Ch6(notification·over-alert 회피), Ch7(통합), Ch8(배포·onboarding·지속 개선).
- 많은 기업이 품질 관리 없는 data factory를 운영 중이며 legacy 방법은 현대적 data 양에 실패. ML 기반 자동 monitoring이 중요 data 깊숙한 issue를 선제 탐지·설명해 ML model·dashboard·의사결정을 보호. 저자는 data quality 민주화를 믿어 솔직히 공유했다.

---

## Summary (핵심 정리)
- 운영은 build vs buy → configuration → enablement → 지속 개선 단계를 따른다.
- build는 통제력, buy는 빠른 시작·낮은 총비용이 강점이며, deployment model(SaaS/in-VPC/hybrid)은 security와 control의 trade-off다(완전한 data 보호는 in-VPC만 보장).
- 비용 효율을 위해 heat 기반으로 중요 table만 deep monitoring하고 최신 data만 query하며, config는 API와 UI를 함께 제공한다.
- kickoff·live training·on-demand·office hours·ongoing support로 onboarding하고 silent failure를 막는다.
- initiative·triage SLA·executive dashboard·scorecard로 data quality를 지속적으로 측정·개선한다.
