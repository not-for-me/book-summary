# 08. Democratizing Data Quality

## 챕터 개요 (3줄 요약)

- 데이터 품질은 기술적 과제일 뿐 아니라 문화적 과제로, 전사적 buy-in이 있어야 데이터 신뢰가 달성됨을 강조한다.
- "데이터를 제품처럼(data-as-a-product)" 다루는 패러다임, 데이터 플랫폼 신뢰 구축, 품질 소유권(RACI) 배정, 데이터 인증(certification)을 다룬다.
- 데이터 리터러시·거버넌스·컴플라이언스 강화와 Toast의 팀 구조 진화 사례로 데이터 품질 전략 수립법을 제시한다.

---

## 1. Treating Your "Data" Like a Product (데이터를 제품처럼)

> 데이터는 더 이상 2등 시민이 아니며, 가장 선도적인 팀은 데이터를 제품으로 취급한다.

- 데이터 제품이 제공할 결과: 접근성·민주화 향상, 빠른 ROI, 시간 절약, 정밀한 인사이트.
- 데이터 제품의 품질: 신뢰성·옵저버빌리티("몇 개의 9"), 확장성, 확장 가능성(extensibility), 사용성, 보안·컴플라이언스, 릴리스 규율·로드맵.

### Perspectives (사례)
- Convoy(Chad Sanderson): 데이터를 "서비스"(파이프라인 전체가 제품) 또는 "출력물"(코드베이스의 산출물, 예: 웨어하우스 SQL 코드베이스)로 취급. 둘 다 테스트·SLA·문서화·모니터링 필요.
- Uber(Atul Gupte): "데이터 제품 매니저(Data Product Manager)" 역할 등장 — 데이터 민주화·time to value 책임, 내부 도구·플랫폼 구축.

### Applying the Approach (5가지 적용법)
- 이해관계자 정렬을 조기·자주 확보(data storytelling으로 투자 정당화).
- 제품 관리 마인드셋 적용(스코핑 문서·스프린트, 확장성·미래 사용 고려, 비즈니스 목표 연계 KPI).
- 셀프서비스 도구 투자(데이터 메시의 핵심 원칙, 비기술 팀 자가 충족).
- 데이터 품질·신뢰성 우선(데이터 신뢰성 성숙도 곡선: Reactive→Proactive→Automated→Scalable).
- 적합한 팀 구조 탐색(hub and spoke 모델 선호: 중앙 플랫폼 팀 + 분산 분석가).

```
   Data Reliability Maturity Curve:
   Reactive  -> Proactive -> Automated -> Scalable
   (firefight) (manual QA)  (scheduled)  (DevOps,
                                          anomaly detection)
```

---

## 2. Building Trust in Your Data Platform (플랫폼 신뢰 구축)

> 데이터 플랫폼을 제품처럼 다루려면 비즈니스 목표와 정렬하고 올바른 이해관계자의 buy-in을 얻어야 한다.

- 제품 목표를 비즈니스 목표와 정렬(어떤 데이터가 수익을 견인하는가, 누가 언제 접근하는가, 컴플라이언스는?).
- 올바른 이해관계자의 피드백·buy-in 확보(리더십에 비전 판매 + 실사용자에게 일상 사용 사례 + 고객 중심 페르소나 접근).
- 단기 이익보다 장기 성장·지속가능성 우선(Uber 5년, LinkedIn은 2008년부터 반복 구축).
- 데이터 기준 메트릭과 측정법 합의(SLO·SLI, 데이터 다운타임 시간 등).
- Build vs Buy 판단: 보편적 과제(웨어하우스·레이크·시각화)는 구매, 틈새·전략적 과제는 구축.

---

## 3. Assigning Ownership for Data Quality (품질 소유권 배정)

> 데이터가 깨질 때 하류 영향 범위를 "blast radius(폭발 반경)"라 하며, 여러 페르소나가 관여한다.

- 주요 페르소나: CDO(Ophelia), BI 분석가(Betty), Analytics Engineer(Anna, dbt), Data Scientist(Sam), Data Governance Lead(Gerald), Data Engineer(Emerson), Data Product Manager(Peter).
- 데이터 신뢰성은 궁극적으로 모두의 책임. RACI(Responsible, Accountable, Consulted, Informed) 매트릭스로 소유권 매핑.
- 대규모(Netflix/Uber)는 데이터 엔지니어·DPM이 모니터링·알림 담당, 그 외엔 주로 엔지니어·PM이 책임. 나쁜 선택의 피해는 BI 분석가가 짊어짐.

---

## 4. Creating Accountability & Certifying Your Data (책임성·데이터 인증)

> 데이터 엔지니어는 데이터 카탈로그가 아니다. 접근성과 신뢰의 균형을 위해 데이터 인증이 필요하다.

- 문서 버전 관리 문제(V6_Final_RealFinal)처럼 중복 모델 난립 → 비효율·낮은 신뢰·다운타임 증가.
- 데이터 디스커버리(data discovery): 카탈로그의 "이상적 상태"가 아닌 실시간 현재 상태를 도메인별로 제공.
- 데이터 인증(certification): 품질·옵저버빌리티·소유권·이슈 해결·소통의 SLA를 충족한 자산을 전사 사용 승인.

### 인증 프로그램 7단계
- Step 1: 데이터 옵저버빌리티 역량 구축(인시던트 대시보드로 이상·스키마 변경 자동 표면화).
- Step 2: 데이터 소유자 결정(라이프사이클 각 계층별 책임자).
- Step 3: "좋은 데이터" 정의(freshness/distribution/volume/schema/lineage/downtime/query speed/ingestion KPI).
- Step 4: 핵심 데이터셋에 명확한 SLA/SLO/SLI 설정(구체적·측정가능·달성가능, 미달 시 대응 포함). 가장 중요한 테이블부터, bronze/silver/gold 등급제.
- Step 5: 소통·인시던트 관리 프로세스 수립(Slack/PagerDuty/Teams).
- Step 6: 인증 태깅 메커니즘(탈중앙화 권장).
- Step 7: 팀·소비자 교육(alert fatigue 주의).

---

## 5. Case Study: Toast (팀 구조 진화)

> Greg Waldman이 5년간 Toast 데이터 팀을 1명→20+명으로 키우며 중앙→분산→하이브리드를 오간 사례.

- 초기(200명): Excel 운영 한계 → Greg 합류, 도구·프로세스 구축.
- 성장(400~850명): 중앙 팀이 수요 못 따라감 → 부서별 분산 분석가 자생, 모던 분산 스택(S3/Airflow/Snowflake/Stitch/Looker) 이전.
- 재중앙화(1,250명+): 데이터 일관성·소통 위해 분석가를 다시 통합(Finance & Strategy 부서).
- 조언: 데이터 제너럴리스트 채용(단, 데이터 엔지니어는 예외로 일찍 채용), 다양성 우선, 과잉소통, "단일 진실 공급원" 과대평가 금지(80/20, 방향성 정확도면 충분), 소통 능력 좋은 사람 채용.

```
   Centralized  <-->  Hybrid (hub & spoke)  <-->  Decentralized
   (consistency)      (central platform +         (domain expertise,
                       embedded analysts)          but silos)
   -> Business needs drive the structure; stay agile
```

---

## 6. Increasing Data Literacy (데이터 리터러시 향상)

> 데이터 리터러시는 데이터를 읽고 쓰고 소통하여 가치를 창출하는 능력으로, 품질 문화의 출발점이다.

- 하향식 buy-in + 상향식 채택: 셀프서비스 도구·교육으로 데이터를 접근 가능하게.
- "Head of Data Literacy" 역할로 부서별 데이터 스킬(Excel/SQL/R/Python) 스코어카드·목표 설정.
- 최대 장애물: 문서화 부족 → 지식 공유를 조기·자주. 도구: 데이터 카탈로그, DBMS, 데이터 모델링 도구, 운영 분석 대시보드.

---

## 7. Prioritizing Data Governance and Compliance (거버넌스·컴플라이언스)

> 데이터 거버넌스는 가용성·사용성·출처·보안 관리로, GDPR·CCPA 시대에 핵심이나 80% 이상이 실패한다(Gartner).

- 전통적 수동 카탈로그는 클라우드 스택 속도를 못 따라감 → ML·지식그래프 기반으로 진화.
- 자동 카탈로그 3종: In-house(Uber Databook, 커스터마이징 우수하나 가시성·비용 문제), Third-party(ML 기반, UI 사용성 이슈), Open source(Lyft Amundsen, Apache Atlas/Magda/CKAN — 수동 태깅 부담).
- 카탈로그를 넘어: 리니지·옵저버빌리티로 거버넌스 갭 보완, 자동·분산 정책 시행(PII 식별·접근 제어). 결국 문화가 핵심.

---

## 8. Building a Data Quality Strategy (전략 수립)

> 기술·프로세스·조직 요건을 종합하여 처음부터 데이터 품질 전략을 구축한다.

- 리더십에 품질 책임 부여(측정법·KPI·교차기능 참여·책임자 투명화).
- 데이터 품질 KPI 설정(completeness/freshness/accuracy/consistency/validity 같은 구체적 지표, 모호한 점수 회피).
- 데이터 거버넌스 프로그램 주도(data quality champion, 단기 성과로 추진력 확보).
- 리니지·거버넌스 도구 자동화(수동 모니터링은 한계).
- 양방향 커뮤니케이션 계획 수립(리더십·이해관계자·데이터 스튜어드 정렬).

---

## Summary (핵심 정리)

- 데이터 민주화는 기술적인 만큼 문화적인 과정이며, RACI 어디에 있든 데이터 품질이 성공의 핵심이다.
- 4단계: 데이터를 프로덕션 SW처럼 다루기, 소스에서 품질을 우선하는 팀 구성, 데이터 리터러시를 일급 시민으로, 거버넌스를 확장하는 프로세스·기술 채택.
- 점점 더 많은 기업이 데이터 신뢰성 엔지니어·옵저버빌리티 전문가·리터러시 책임자를 채용하며 흐름이 바뀌고 있다.
