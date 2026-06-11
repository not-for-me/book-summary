# 03. Assessing the Business Impact of Automated Data Quality Monitoring

## 챕터 개요 (3줄 요약)
- automated data quality monitoring 투자 가치는 your data, industry, data maturity, stakeholders 네 factor로 self-assessment할 수 있다.
- 4V(volume, variety, velocity, veracity)와 data 구조(structured/semistructured/unstructured), update cadence가 적절한 monitoring 전략을 결정한다.
- ROI 분석은 incident 빈도·비용·절감 시간 같은 quantitative 측면과 trust·collaboration 같은 qualitative 측면을 함께 고려해야 한다.

---

## 1. Assessing Your Data
> IBM의 Four V's(volume, variety, velocity, veracity) framing으로 data 특성이 automated monitoring 적합성을 알려준다.

- **"We don't have data quality issues"**: 거의 항상 둘 중 하나 — 아무도 data를 안 쓰거나(issue 미탐지), data 생성 system을 아무도 안 바꾸거나(정적). 대부분 issue는 존재하나 신경 안 쓸 뿐.

### 1.1 Volume
> 소량이면 manual 가능하나, millions~billions row가 되면 작지만 중요한 segment의 issue 탐지가 핵심 과제가 된다.

- 수십 record/day라도 automated monitoring 가치 있음. 대규모에선 unsupervised model + alert fatigue 방지 notification 필수.

### 1.2 Variety
> data variety가 클수록 issue risk surface도 커진다(구조·수집·시간·갱신·entity·관계·granularity 등).

- column 유형별 monitoring 관심사: **identifier**(uniqueness, format, relational integrity), **time**(granularity, sequencing, interarrival), **segment**(validity, distribution, cardinality), **metric**(average, distribution, outlier).
- **Unstructured data**(video/image/audio/text): metadata 모니터링하거나, ML classifier(blurry image 탐지), 또는 static deep learning model의 **embedding drift**를 unsupervised로 감시(retrain 시 embedding이 크게 이동하므로 static model에서만 유효; foundation model은 버전 간 안정적).
- **Semistructured data**(주로 JSON): schema validation + custom algorithm + rule 혼합. JSON **object**는 column으로 확장(`json.name`, `json.age`), JSON **array**는 normalize하면 다른 정보와 격리되거나 sparse column 폭증 → array element를 random sampling해 monitoring하는 전략 권장.
- **Structured data**(가장 중요): ① **normalized relational**(OLTP, issue의 root지만 좁은 surface, join 비용 큼), ② **fact tables**(denormalized, 가장 통찰적·business value, ML+rule+metric 혼합), ③ **summary tables**(dashboard용 aggregate, 최신값만 보이므로 snapshot 필요).

### 1.3 Velocity
> data update cadence에 따라 적절한 monitoring 전략이 달라진다.

- monthly보다 드문 data(census, financial): 대개 manual. model이 학습할 history가 부족하고 생성 process가 바뀌어 무의미.
- weekly/daily/hourly: unsupervised ML이 강력 — history 충분, sample size 충분, manual 비용 과다.
- minute/millisecond(fraud, credit approval): real-time programmatic 대응 필요 → false positive 없는 **deterministic rule-based testing**만 사용.

### 1.4 Veracity
> data의 truthfulness — 직접 알 수 없으므로 new issue introduction risk와 SLA 등 신뢰 요인으로 근사한다.

- 가장 veracity 낮은 source: third-party data(예고 없이 변경), 복잡한 system 간 상호작용, 지속 변경·개선되는 system, legacy system.
- **Special cases**(unsupervised ML 어려움 → manual/rule): 입력 시점 검증이 중요한 data(수기 주소), entity·transaction 수가 매우 적은 data(M&A), 단일 static dump(pharma trial).

---

## 2. Assessing Your Industry
> financial services, ecommerce, media, technology, real estate, healthcare에서 data quality imperative가 특히 절실하다.

### 2.1 Regulatory Pressure
> 규제가 일부 조직의 automated data quality 투자를 강제한다.

- financial institution은 규제 미충족 시 거액 벌금(Wells Fargo $250M, Citibank $400M). EDM Council의 **CDMC framework**가 sensitive data best practice 제공 — control metric 자동 모니터링·alert(1.1), data catalog ownership(1.2), data quality 측정·전달(5.2), data lineage(6.0). 본서 기법 없이 scale 구현은 상상하기 어려움.
- 규제는 긴급 사례에서 시작해 점차 표준이 됨(손 씻기 비유). AI/ML 확산으로 data quality 규제는 더 늘어날 전망.

### 2.2 AI/ML Risks
> model은 training data와 production data 불일치 시 오작동하며, 종종 완전히 멈추지 않고 특정 segment에서만 나빠져 debug가 어렵다.

- **Feature shocks**: feature가 하루에 historical norm 밖으로 급등. training 시 feature importance 약화. production에서 linear model은 비례 확대, tree-based는 극단으로 routing, neural network는 예측 불가하게 erratic.
- **NULL increases**: 최선엔 feature shock과 동일, 최악엔 NULL/zero 혼동이나 aggregation 시 NULL 전파(item price 1/10 NULL → order 평균이 NULL).
- **Change in correlation**: ID 생성 실패로 credit score가 잘못된 loan에 join되면 분포는 같아도 상관 소실. linear(특히 비정규화 시 민감)·tree-based(다른 경로 routing)·neural network 모두 큰 영향.
- **Duplicate data**: overfitting 유발. train/test split에 중복 record가 양쪽에 들어가면 model이 실제보다 성능 좋아 보임(memorization).

### 2.3 Data as a Product
> data를 판매·패키징하면 data 품질이 곧 product 품질이라 monitoring 가치가 커진다.

- 직접 판매(credit aggregator, MLS, 재무 data, 경쟁 가격)뿐 아니라, media platform이 creator에게 제공하는 view count·watch time처럼 product의 핵심인 경우도 포함.

---

## 3. Assessing Your Data Maturity
> Monica Rogati의 "Data Science Hierarchy of Needs" pyramid 상에서 조직 위치가 투자 시점을 알려준다.

- 초기엔 observability 질문(ingest됐나? 갱신 시점?). pyramid 중간(explore·transform) 단계가 automated monitoring 도입 적기 — 이후 aggregation·optimization이 high-quality data에 의존.
- "issue를 고칠 resource가 없으니 monitoring도 안 한다"는 흔한 오판 — issue는 이미 사람들을 방해하며 debt를 키움. issue를 **식별하고 공격적으로 우선순위화**하는 근육을 길러야 함(소수 critical fact table부터, fix 기준 사전 합의).
- **data stack이 maturity 지표**: cloud data warehouse 미도입이면 대개 시기상조. production DB 직접 모니터링은 부하·production 영향, cloud storage(S3)는 metadata만 가능 → lakehouse(Presto/Athena/Databricks)나 warehouse(BigQuery/Snowflake) external table 활용(단 매 run마다 전체 file read라 적재가 더 저렴·고성능일 수 있음). Airflow/dbt 사용, data catalog 표준화도 maturity·monitoring 필요의 신호.

---

## 4. Assessing Benefits to Stakeholders
> 조직이 pyramid를 오를수록 data team이 커지고 monitoring solution에 더 많은 것을 요구한다.

- **Engineers**(data/analytics/platform): freshness·volume·source 간 reconciliation 관심. 쉬운 config, 자동화, 통합 중시 → robust API(직접 호출/Python, Airflow·Databricks operator, Git 관리 YAML/JSON config).
- **Data Leadership**: 전체 health·trend·engagement 관심. KPI — coverage(check 정의된 table 수), data arrival times(SLA), trends(time series·WoW), repeat offenders(가장 문제 많은 table).
- **Scientists**(analyst/scientist/ML platform): missing·duplicate·distribution change 관심. API보다 **사용 쉬운 UI**와 root cause 탐색용 rich visualization 선호.
- **Consumers**(product/ops/marketing/compliance): 가장 깊은 domain SME. single source of truth와 명확하고 적절한 urgency의 notification 필요(해결엔 직접 관여 안 함).

---

## 5. Conducting an ROI Analysis
> quantitative와 qualitative metric을 모두 고려해 투자 ROI를 평가한다.

### 5.1 Quantitative Measures
> issue 빈도·건당 비용·미탐지 비율 × table 수로 incident 비용을 추정하고, 절감되는 작업 시간을 계산한다.

- 작업 시간 절감 대상: check 생성·유지, notification 설정, 조사·root cause, metric 모니터링. trial로 실측 권장.
- **20,000 table 예시 분배**: temporary 10,000(50%, 모니터링 안 함), processing 9,000(45%, table observability — freshness·volume), important 900(5%, data quality ML), key 90(0.5%, +metric/segment), critical 10(0.1%, +record-level validation rule). 대부분 노력은 소수 중요 table에 집중. 총 설정 1,390시간, triage 연 3,175시간 수준.
- licensing·infra·computation 비용과 예산 예측가능성도 고려.

### 5.2 Qualitative Measures
> trust·collaboration 같은 정성적 이득과, 변화 저항·유지 부담 같은 정성적 단점을 함께 본다.

- 장점: 개발 cycle 가속, partner 신뢰, audit trail, data quality 민주화·collaboration, data 전반 ROI multiplier.
- 단점: 변화 저항·재교육, security risk(VPC 미배포 SaaS), in-house 유지 부담, false positive로 인한 alert fatigue.
- data leader는 data quality를 전체 data budget의 ROI multiplier로 봄 — 예: $100M 투자(20% 기대수익 $120M)에서 품질 10% 개선 시 $132M, 연 +$12M. 많은 대기업은 ROI 분석조차 생략 — 가치가 자명해 관건은 속도.

---

## Summary (핵심 정리)
- automated monitoring 적합성은 data, industry, data maturity, stakeholders 네 factor로 self-assess한다.
- 4V(volume·variety·velocity·veracity)와 data 구조·cadence가 unsupervised ML·rule·manual 중 적절한 전략을 결정한다.
- regulatory pressure, AI/ML risk(feature shock·NULL·correlation·duplicate), data-as-a-product가 industry별 절박함을 키운다.
- cloud warehouse·dbt·data catalog 등 data stack이 maturity와 도입 시점을 알려주며, 도입 적기는 pyramid 중간 단계다.
- ROI는 incident 비용·절감 시간(quantitative)과 trust·collaboration(qualitative)을 함께 고려하며, 종종 data budget 전체의 multiplier로 평가된다.
