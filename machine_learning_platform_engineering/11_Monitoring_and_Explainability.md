# 11. Monitoring and Explainability

## 챕터 개요 (3줄 요약)

- production model의 신뢰성을 위해 basic(operational) monitoring과 data drift monitoring을 구축하고 explainability로 결정 과정을 이해한다.
- BentoML 대시보드·custom metric·Loki 로깅·Prometheus Alertmanager로 관측성과 alerting을 완성한다.
- image는 deepchecks·Eigen-CAM, tabular/recommender는 deepchecks FeatureDrift·EMF로 drift 감지와 설명을 수행한다.

---

## 11.1 Monitoring

> 모든 production app은 monitoring 없이 완성될 수 없다 — 조기 감지로 downtime·SLA 위반 최소화.

- monitoring 2축: **basic**(operational, SLA: uptime/throughput/latency/quality, error budget) + **data drift**(input과 target 관계의 일관성).

### 11.1.1 Basic monitoring

- 2 category: **resource utilization**(memory/CPU/pod 수 — 제한 있음, 최적 할당) + **request tracking**(latency, non-200 status code).
- BentoML은 `/metrics` 노출 → Prometheus **PodMonitor**(특정 pod scrape 정의)로 수집 확인. Grafana에 BentoML **prebuilt dashboard.json** import(in-progress requests, RPS, success rate, CPU/memory).

### 11.1.2 Custom metrics

- default metric으로 부족한 business logic 추적. `pip install prometheus-client`.
- object detection: **Histogram**(confidence score, decile bucket) → `histogram.observe(confidence)`. Grafana Gauge + PromQL `histogram_quantile(0.9, ...)`.
- recommender: **Counter**(ranked_movies 존재 횟수, 증가만/reset). Grafana time series.

### 11.1.3 Logging

> metric(정량)과 달리 log는 error message·stack trace 등 정성적·context 정보 제공 → root cause 파악.

- BentoML logger(일반 Python logger): format 설정 → `getLogger("bentoml")` → level. 예: 결과 0개 시 `bentoml_logger.error(...)`.
- **centralization** 이점: 단일 플랫폼, 다중 소스 aggregation, 팀 협업. **Loki**(Grafana Labs, 경량 log aggregation, metadata만 indexing해 저비용·확장). Prometheus·Grafana 통합으로 log↔metric 상관.

### 11.1.4 Alerting

> 문제(malfunction·성능 저하·security)를 자동 통지 → 선제 대응, downtime 감소.

- **Alertmanager**(Prometheus 생태계): Prometheus가 alert 생성 → Alertmanager가 rule 따라 email/Slack/PagerDuty로 라우팅.
- alert rule = PromQL 조건. `up` metric(1=up, 0=down), `absent()`(metric 부재 시 1, deployment 삭제 감지).
- rule에 `for`(지속 시간, 5m), `labels`(severity), `annotations`(summary/description). Prometheus serverFiles에 alert group 추가 → helm upgrade.
- alert 상태: green(미발동) → yellow(pending, 조건 충족 but for 대기) → red(triggered).
- Alertmanager config: SMTP(Gmail app password) global + route(group_by, group_wait, repeat_interval) + receivers. severity별 라우팅(critical→PagerDuty, low→Gmail).

```yaml
- alert: MissingUpMetric
  expr: absent(up{job="yatai/bento-deployment", ...}) == 1
  for: 5m
  labels: { severity: critical }
  annotations: { summary: "...", description: "..." }
```

---

## 11.2 Data drift detection

> 6장(income classifier, tabular)을 object detection·recommender로 확장. **deepchecks** 라이브러리 활용.

- deepchecks 기능: data integrity check, model validation, data drift detection, customizable check.

### 11.2.1 Object detection (image)

- tabular과 달리 이미지는 정해진 feature가 없음 → image property(brightness, contrast, aspect ratio, area) 비교.
- 실습: PIL `ImageEnhance.Brightness`로 test image brightness 조정해 drift 유발 → `IDCardDataset`(torchvision VisionDataset subclass) + `load_dataset`(deepchecks VisionData 반환).
- **deepchecks ImagePropertyDrift** 실행 → KS Test drift score. brightness만 높은 drift(0.62), Area·Aspect Ratio는 0. HTML 리포트로 분포 시각화.
- production: 처리된 image·label을 저장 후 주기적으로 drift score 계산.

```python
check_result = ImagePropertyDrift().run(train_dataset, test_dataset)
# Brightness drift score 0.62 (변경됨), Area/AspectRatio 0.0
```

### 11.2.2 Movie recommender (embeddings)

- matrix factorization은 신규 user/item 대비 재학습 필요. **user·item factor(embedding)** 분포 변화로 preference/popularity shift 감지.
- 실습: 일부 movie rating을 +1(≤5 유지)로 drift 유발 → drifted/non-drifted 두 dataset으로 재학습 → embedding 추출.
- **deepchecks FeatureDrift**(개별 feature drift): matrix transpose(컬럼=item/user) → pandas → Dataset. item latent factor 분포 drift 확인 → 특정 movie preference 변화. latent factor 모니터링 pipeline 구축.

---

## 11.3 Explainability

> monitoring이 "잘 작동하나"라면, explainability는 "왜 그렇게 결정했나" — 신뢰·debugging·규제·성능 개선.

- **XAI(interpretable AI)**: 복잡한 model 결정을 인간이 이해 가능하게. bias 식별, 강약점 파악, 관련 feature 기반 판단 확인.
- business: stakeholder 신뢰, 규제(금융 credit risk의 ECOA — 거부 사유 설명 의무), 의사결정 지원.
- 2 유형: **model-based**(linear regression·decision tree 등 투명 구조) vs **post hoc**(학습 후 black box 분석 — SHAP, LIME).

### 11.3.1 Object detection (Eigen-CAM)

- **Eigen-CAM**(CAM 계열): CNN 결정에 대한 heatmap 시각화(영향력 큰 영역). 재학습 불필요, 복잡한 architecture에 적용.
- ID card 예: model이 face에 집중하는지 확인 → face 무관하면 face 없는 ID card로 일반화 가능.
- `EigenCAM(model, target_layers, task='od')` → grayscale_cam → `show_cam_on_image`. model이 ID card·모서리에 집중 확인. 재학습 시 검증에도 활용.

```python
cam = EigenCAM(model, target_layers, task='od')
grayscale_cam = cam(rgb_img)[0, :, :]
cam_image = show_cam_on_image(img, grayscale_cam, use_rgb=True)
```

### 11.3.2 Movie recommender (EMF, model-based)

- **EMF(explainable matrix factorization)**: latent space의 유사 user/item 식별로 설명. explainability score = (item을 rate한 유사 user 수)/(전체 rate user 수)를 학습 weight로 사용.
- `EMFModel` fit → `Recommender` wrap → `EMFExplainer.explain_recommendations()`. 각 user top-10 추천 + explanations 컬럼({rating: 유사 user 수}).
- 예: "9명 유사 user 중 8명이 4점 이상" → user·business 신뢰↑. 유사 user 없으면 explanation 비어있음.

---

## Summary (핵심 정리)

- ML app monitoring은 서비스 신뢰성·성능 유지에 핵심 — basic monitoring은 resource·request metric을 BentoML prebuilt dashboard로 시각화한다.
- custom metric(confidence score, ranked movie count)으로 app-specific 세부를 추적해 dashboard에 통합한다.
- logging은 debugging용 context를 제공하며 Loki로 centralize해 관측성을 높인다.
- alerting은 선제 대응에 필수 — alert rule + Alertmanager 라우팅으로 적시 대응한다.
- deepchecks로 image·embedding drift를 감지하고, Eigen-CAM·EMF로 model 결정을 설명해 투명성·책임성을 높인다. 다음 챕터(12장, Part 4)에서 LLM-powered system을 다룬다.