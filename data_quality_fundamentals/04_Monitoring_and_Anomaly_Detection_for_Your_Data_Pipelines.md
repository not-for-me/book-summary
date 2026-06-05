# 04. Monitoring and Anomaly Detection for Your Data Pipelines

## 챕터 개요 (3줄 요약)

- 테스트로 잡을 수 없는 "unknown unknowns"를 모니터링과 이상 탐지(anomaly detection)로 커버하는 방법을 다룬다.
- 데이터 옵저버빌리티의 5대 기둥(freshness, volume, distribution, schema, lineage)을 SQL/Python으로 직접 구현하는 튜토리얼을 제공한다.
- 머신러닝으로 이상 탐지를 확장하고, precision/recall/F-score로 false positive와 false negative의 트레이드오프를 균형 잡는 법을 설명한다.

---

## 1. Knowing Your Known Unknowns and Unknown Unknowns (알려진/알려지지 않은 미지수)

> 예측 가능한 이슈(known unknowns)는 테스트로, 예측 불가능한 이슈(unknown unknowns)는 모니터링·이상 탐지로 대응한다.

- Known unknowns: null 값, 특정 freshness 이슈, 정기 시스템의 스키마 변경 등 테스트로 예측 가능.
- Unknown unknowns: 분포 이상, 6→600 컬럼 폭증, ETL 변경으로 누락된 테스트, 미감지된 stale 데이터, data drift 등.
- "좋은 데이터"의 모습을 이해하면 "나쁜 데이터"를 사전에 식별하기 쉬워진다.

---

## 2. Building an Anomaly Detection Algorithm (이상 탐지 알고리즘 구축)

> 거주가능 외계행성(EXOPLANETS) 모의 데이터로 SQLite·Python·Jupyter를 활용해 freshness·distribution 탐지기를 직접 만든다.

### Monitoring for Freshness
- DATE_ADDED 컬럼을 기준으로 일일 신규 행 수를 집계하여 업데이트 패턴 파악.
- DAYS_SINCE_LAST_UPDATE 메트릭(JULIANDAY + LAG)으로 "데이터가 며칠 묵었는가"를 계산.
- 임계값(threshold) 파라미터가 쿼리를 "탐지기"로 전환한다: 작으면 recall↑(가짜 경보 多), 크면 precision↑(놓침 多).
- 최적 파라미터는 너무 낮지도 높지도 않은 "Goldilocks Zone"에 위치한다.

### Understanding Distribution
- 분포는 데이터의 기대값과 빈도를 알려준다(예: null 비율 10%→90% 변화 감지).
- 중심극한정리(Central Limit Theorem): 충분히 무작위인 표본 평균의 분포는 정규분포(Gaussian)에 근사.
- 단순 z-score(표준점수) 기반 탐지는 한계: BI 데이터는 상관·교란이 많고(예: 일요일 휴업), "이상"과 "흥미로운" 관측은 다르다.
- null 비율 spike 탐지: 단순 임계값(>0.9) → 중복 알림 필터링 → 2주 rolling average 적용으로 정밀도 향상(증가만 탐지).

---

## 3. Building Monitors for Schema and Lineage (스키마·리니지 모니터)

> 좋은 옵저버빌리티는 이상 탐지뿐 아니라 맥락(상류 원인, 하류 영향)도 제공한다. 스키마와 리니지가 이를 담당한다.

### Anomaly Detection for Schema Changes
- EXOPLANETS_EXTENDED는 2개 필드(ECCENTRICITY, ATMOSPHERE)가 추가된 상위집합 → 스키마 변경 사례.
- DB는 버전 이력이 없으므로 EXOPLANETS_COLUMNS 테이블로 날짜별 컬럼 목록을 "버전 관리".
- LAG로 이전 컬럼 목록과 비교하여 변경 발생일(2020-07-19) 식별.

### Visualizing Lineage
- 리니지는 5대 기둥 중 가장 총체적: (1)영향받는 하류 소스, (2)근본 원인 상류 소스를 알려준다.
- HABITABLES 테이블은 EXOPLANETS에 의존 → 의존성 그래프(dependency graph) 형성.

### Investigating a Data Anomaly (근본 원인 추적)
- HABITABLES의 평균 habitability가 0.5→0.25로 급락(분포 이상) 발견.
- null 비율은 정상 → zero 비율을 점검하니 2020-07-19부터 급증.
- 같은 날 EXOPLANETS_EXTENDED 스키마 변경 발생 → 상류 변경이 하류 zero rate spike의 근본 원인으로 연결(리니지의 가치).

```
   EXOPLANETS / EXOPLANETS_EXTENDED  (source)
            |                |
   EXOPLANETS_COLUMNS    HABITABLES   (downstream products)
   (schema versioning)  (habitability)
            schema change 2020-07-19
                 --> zero rate spike in HABITABLES
```

---

## 4. Scaling Anomaly Detection with Python and Machine Learning (ML로 확장)

> 머신러닝 탐지기는 수많은 테이블에 유연하게 적용되고, 실시간 학습·복잡한 계절성(seasonality)을 포착한다.

### False Positives and False Negatives
- 이상 탐지는 비지도(unsupervised) 작업이지만, 벤치마킹을 위해 지도 용어(FP/FN)를 사용한다.
- False positive(늑대 소년): 정상인데 경보 → 신뢰 저하. False negative(잠든 경비견): 문제인데 미경보 → 진짜 문제 방치.
- FP 감소는 FN 증가를 수반(트레이드오프) → 중간 지점을 노려야 한다.

### Precision and Recall
- Precision = TP / (TP + FP): 경보가 얼마나 정확한가(믿을 만한가).
- Recall = TP / (TP + FN): 실제 이상 중 얼마나 잡았는가(의존 가능한가).
- 둘은 트레이드오프 → F-score로 통합. Fβ는 "recall을 precision보다 β배 중요하게" 가중(β=1이면 동등=F1).

```
              Predicted
            Neg        Pos
   Actual Neg  TrueNeg   FalsePos
          Pos  FalseNeg  TruePos

   Precision = TP / (TP+FP)
   Recall    = TP / (TP+FN)
   F1 = 2 / (1/Precision + 1/Recall)
```

### F-Scores 실전 예시
- 하와이 미사일 오경보(2018) 사례: 오경보(false positive)보다 미경보(false negative)가 치명적 → recall이 precision보다 중요 → F2/F3 사용.
- AIDS 예시: 무조건 "No" 예측해도 정확도 99.6% → accuracy는 오해의 소지가 있음, precision/recall이 더 유의미.
- 임계값 3일→F1 0.75, 5일→precision 1.0이나 recall 0.5(F1 0.667). 4일에서 F1 최고 → "sweet spot".

### ML 프레임워크
- Facebook Prophet(계절성 예측), TensorFlow(Keras autoencoder), PyTorch(학계 선호), scikit-learn(ARIMA·k-NN·isolation forest).
- MLflow(실험 추적·모델 레지스트리·재현성), TensorBoard(loss·confusion matrix 시각화).

---

## 5. Beyond the Surface: Other Useful Approaches (기타 유용한 접근법)

> 실시간 탐지·적시 알림·예방 정보 제공이 좋은 탐지기의 3대 요건이다.

- Rule definitions/hard thresholding: 명시적 컷오프, 확장성 우수, 잘 정의된 SLA에 적합.
- Autoregressive models: 이전 타임스텝으로 회귀 예측 → ARIMA(AutoRegressive Integrated Moving Average).
- Exponential smoothing: 추세·계절성 제거(Holt-Winters).
- Clustering: k-NN·isolation forest로 "튀는 점" 탐지.
- Hyperparameter tuning: 모델/알고리즘 하이퍼파라미터(학습률, epoch 수 등) 조정.
- Ensemble model framework: 여러 기법을 다수결(majority-voting)로 결합.

---

## 6. Designing Monitors for Warehouses Versus Lakes (웨어하우스 vs 레이크 모니터 설계)

> 구조화된 웨어하우스와 야생의 데이터 레이크는 엔트리포인트 수, 메타데이터 수집·저장·접근 방식이 다르다.

- 레이크는 엔트리포인트가 많아 이질성이 높음 → "one-size-fits-all" 모델 지양, 엔드포인트별 모델/앙상블 권장.
- 레이크 메타데이터는 전처리(타입 강제, 스키마 정렬, 피처 증강)가 더 필요할 수 있음 → 필요시 중간 ELT 단계 설계.

---

## Summary (핵심 정리)

- 어떤 이상 탐지 문제에도 완벽한 분류기는 없으며, FP와 FN(=precision과 recall) 사이엔 항상 트레이드오프가 존재한다.
- 최적 Fβ score 선택이 이 트레이드오프의 가중치를 결정하므로, 자신의 문제에서 무엇이 더 중요한지 정의해야 한다.
- 모델 정확도 논의는 비교할 ground truth가 있어야 의미가 있으며, 다음 장에서 SLA·SLI·SLO로 신뢰성 아키텍처를 확장한다.
