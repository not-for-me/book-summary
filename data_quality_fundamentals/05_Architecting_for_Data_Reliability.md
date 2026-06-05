# 05. Architecting for Data Reliability

## 챕터 개요 (3줄 요약)

- 데이터 신뢰성(data reliability)을 라이프사이클 전 단계에 의도적으로 설계하는 방법을 ingestion·파이프라인·다운스트림으로 나눠 다룬다.
- 6개 계층의 데이터 플랫폼 구축, 데이터 옵저버빌리티, 데이터 다운타임 비용(ROI) 계산법을 설명한다.
- DevOps에서 차용한 SLA·SLI·SLO 프레임워크로 신뢰성을 정의·측정·추적하는 법과 Blinkist 사례를 제시한다.

---

## 1. Measuring and Maintaining High Data Reliability at Ingestion (수집 단계의 신뢰성)

> 수집 시점에 품질 이슈를 잡으면 저품질 데이터가 하류로 퍼지는 것을 최소화한다("garbage in, garbage out").

- 모범 사례: 데이터 정제(cleaning), 데이터 랭글링(wrangling), 데이터 테스트로 형식·일관성·완전성·신선도·유일성 자동 점검.
- 데이터 보강(enrichment): 1차/3차 데이터를 병합·추가하여 결측·부정확 데이터를 보완.
- 테스트 3종: Unit testing(코드 한 줄 검증, 비즈니스 로직과 glue code 분리), Functional testing(대규모 데이터셋·파이프라인 내), Integration testing(가짜 데이터로 유효성 검증).
- 흔한 점검: null 값, freshness, volume, distribution, missing values.
- 테스트는 "예상된" 이슈만 잡으므로, Ch4의 반응형 모니터링·이상 탐지로 보완 필요.

---

## 2. Measuring and Maintaining Data Quality in the Pipeline (파이프라인 단계)

> SRE(Site Reliability Engineering)의 옵저버빌리티 원칙을 데이터에 적용한다.

- 애플리케이션 옵저버빌리티 3대 기둥: Metrics(수치), Logs(정성 기록), Traces(인과 관련 이벤트).
- 데이터 옵저버빌리티 5대 기둥: Freshness(최신성), Distribution(분포·완전성), Volume(도착량), Schema(구조·변경), Lineage(상·하류 의존성).
- 데이터 다운타임을 SRE의 application downtime처럼 시간 함수로 측정 → 정량적 신뢰성 지표 확보.

```
   App Observability        Data Observability (5 pillars)
   - Metrics                - Freshness
   - Logs            ==>    - Distribution
   - Traces                 - Volume
                            - Schema
                            - Lineage
```

---

## 3. Understanding Data Quality Downstream (다운스트림 품질 이해)

> 데이터가 "나쁘다"는 사실은 보통 분석(analytics) 계층에 도달해서야 드러난다. 이해관계자와 "신뢰 가능한 데이터"의 정의를 정렬하는 것이 핵심이다.

- 추적 수단: 데이터 신뢰성 대시보드(TTD/TTR), SLA, SLI(측정 수치), SLO(목표값), Net Promoter Score(만족도).
- DAMA UK의 6대 품질 차원: Completeness, Timeliness, Validity, Accuracy, Consistency, Uniqueness.
- 데이터 엔지니어 실무 측정: 오류/무관 데이터 비율, null·결측 수, 적시성, 중복률, 일관성, 사용 팀 수 등.

---

## 4. Building Your Data Platform (데이터 플랫폼 구축)

> 데이터 플랫폼은 수집부터 분석까지 데이터를 총체적으로 관리하는 기술 조합으로, 6개 상호연결 계층으로 구성된다.

- (1) Data Ingestion: ETL/ELT의 추출·적재 단계, 다양한 소스의 구조화·비구조화 데이터 수집. 각 단계 테스트 권장.
- (2) Data Storage and Processing: 웨어하우스/레이크/레이크하우스 중 선택, 클라우드 네이티브로 대규모 저장·처리.
- (3) Data Transformation and Modeling: 변환(분석용 raw 데이터 준비)과 모델링(비즈니스 로직을 테이블·관계로)을 구분. 셀프서비스·no/low-code 확산.
- (4) Business Intelligence and Analytics: 대시보드·시각화로 데이터를 실행 가능하게(data storytelling).
- (5) Data Discovery and Governance: 전통 카탈로그의 한계를 data discovery(실시간·도메인별 현재 상태)로 보완.
- (6) Data Observability: 단일 단계가 아닌 라이프사이클 전체를 관통하는 접근.

```
   +--------------------------------------------------+
   | 6) Data Observability (woven throughout)         |
   +--------------------------------------------------+
   1) Ingestion -> 2) Storage/Processing -> 3) Transform/Model
        -> 4) BI/Analytics    5) Discovery & Governance
   (interconnected, not strictly stacked)
```

---

## 5. Developing Trust in Your Data (데이터 신뢰 구축)

> 가장 진보된 데이터 스택도 신뢰할 수 없으면 무용지물이다. 데이터 옵저버빌리티가 신뢰 구축의 첫걸음이다.

### Data Observability
- 라이프사이클 모든 단계에서 데이터 건강을 완전히 이해하는 능력(5대 기둥 기반 자동 모니터링·알림·triage).
- 효과적 도구는 기존 스택에 연결되어 엔드투엔드 리니지·정지 상태(at rest) 모니터링을 제공(데이터 추출 없이 보안 유지).

### Measuring the ROI on Data Quality
- 데이터 엔지니어·과학자는 업무 시간의 40% 이상을 데이터 문제 해결에 소비, 기업은 연 $15M 이상 다운타임 비용 발생.
- TTD(Time To Detection): 이슈 표면화까지 걸린 시간(흔히 일·주·월 단위, 다운스트림 소비자가 먼저 발견).
- TTR(Time To Resolution): 알림 후 해결까지 걸린 시간.
- 데이터 다운타임 비용 = (TTD시간 + TTR시간) × 시간당 다운타임 비용.
- 연간 깨진 데이터 비용 = Labor cost + Compliance risk(GDPR 벌금 등) + Opportunity cost.

---

## 6. How to Set SLAs, SLOs, and SLIs for Your Data (SLA/SLO/SLI 설정)

> DevOps에서 영감을 받아 SLA·SLI·SLO로 데이터 신뢰성을 우선순위화·표준화·측정한다.

- SLA(Service-Level Agreement): 약속과 미달 시 보상(예: Slack 99.99% 가동 보장, 미달 시 크레딧).
- SLI(Service-Level Indicator): 실제 측정하는 구체적 수치.
- SLO(Service-Level Objective): SLI에 대해 설정한 목표값.

### 3단계 절차
- Step 1 — Defining (SLA): "신뢰 가능한 데이터"를 합의·정의. 데이터 인벤토리·과거 성능으로 기준선 확보, 소비자 피드백 수집.
- Step 2 — Measuring (SLI): 인시던트 수(N), 핵심 테이블 갱신 빈도, 기대 분포 등 건강 지표 측정.
- Step 3 — Tracking (SLO): 허용 다운타임 범위 설정(자동 모니터링 없으면 더 관대하게), 심각도 등급화·대시보드 추적.
- 설정 자체로는 개선되지 않으며, 데이터 생산자·엔지니어·분석가·소비자 간 정렬과 실행 책임이 필수.

---

## 7. Case Study: Blinkist

> 실시간 데이터 부재로 마케팅 지출이 감소한 ebook 구독 서비스가 데이터 신뢰성 엔지니어링으로 문제를 해결한 사례.

- COVID-19로 과거 데이터가 현실을 반영하지 못해 실시간 데이터가 필수가 됨(Facebook/Google 캠페인 자동 최적화 의존).
- 팀이 근무 시간의 50%를 데이터 firefighting에 소비, 매주 임원 콜에서 신뢰 회복에 시달림.
- 해결: 데이터 거버넌스·품질·리팩토링에 집중, 데이터 신뢰성을 "first-class citizen"으로 취급, SLA/SLI 추적.
- 성과: 6인 데이터 엔지니어 팀이 주당 120시간 절감, 타겟팅·채널 운영 개선.
- 교훈: SLO 설정·SLI 측정 자체가 아니라, 우선순위화하고 실제 실행하는 것이 성공의 핵심.

---

## Summary (핵심 정리)

- 데이터 신뢰성 아키텍처는 3가지 접근이 필요하다: DevOps식 프로세스(테스트·옵저버빌리티) 선제 투자, 견고한 데이터 플랫폼 구축, 조직 전반의 SLA/SLI/SLO 정렬.
- 이 단계들 없이는 신뢰 가능한 고품질 데이터를 달성하기 어렵다.
- 사일로된 품질 전략을 전사 우선순위로 전환하는 것은 점진적 과정이며("로마는 하루아침에 이뤄지지 않음"), 다음 장은 인시던트 관리·해결을 다룬다.
