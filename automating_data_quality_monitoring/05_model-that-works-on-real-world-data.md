# 05. Building a Model That Works on Real-World Data

## 챕터 개요 (3줄 요약)
- 현실 data는 seasonality, time-based feature, chaotic table, updated-in-place, column correlation 같은 난제를 가지며, 각각에 대한 mitigation 전략이 없으면 model이 심하게 over/under-alert한다.
- human label은 너무 비싸므로, synthetic anomaly("chaos")를 programmatic하게 주입해 탐지하는 것이 real issue 탐지의 좋은 proxy다.
- backtest로 chaos 유무 데이터에 model을 돌려 AUC·precision·recall·F1로 성능을 측정하고 반복 개선한다.

---

## 1. Data Challenges and Mitigations
> 현실 data의 난제를 극복하는 전략이 있어야 model이 noisy하지 않고 가치 있게 된다.

### 1.1 Seasonality
> 인간 행동의 시간·요일·월·연 패턴이 거의 모든 data에 나타나, today vs yesterday 비교만으로는 부족하다.

- 지난주 같은 요일과만 비교해도 문제: issue 지속 기간 불명, 지난주 자체가 anomaly·holiday일 수 있음.
- 해법: **여러 과거 시점(어제, 이틀 전, 1주 전, 2주 전 등)에서 sampling** — 어느 것과 비교해도 정상이면 today는 정상. 또한 metadata 통계에 time series model을 적용해 장기 seasonality를 감지·dampen.

### 1.2 Time-Based Features
> timestamp·ID처럼 시간과 직접 상관된 column은 today 여부를 자명하게 알려주므로 sample에서 제거해야 한다.

- timestamp는 partition time column과의 delta로 encoding해 명백한 상관 제거. 덜 명백한 것들이 문제: autoincrementing ID, "day of month/week" 표현, version identifier.
- 탐지: summary statistic(매일 커지는가)도 일부 가능하나, 전체 dataset으로 간단 model을 별도 구축해 **지속적으로** 예측에 과도하게 significant한 feature를 찾아 제거(일시적 중요 feature 제거 시 실제 issue까지 지울 위험 주의).

### 1.3 Chaotic Tables
> ad hoc 프로세스나 미성숙·급변 제품 때문에 chaotic한 table은 변동성을 반영하지 않으면 over-alert된다.

- table별 severity time series를 구축해 chaos 수준을 학습하고 **동적 threshold**를 설정. 초기엔 매우 보수적(거의 alert 불가능)으로 시작해 exponential decay(예: 10일마다 절반)하며, logging된 score에 fit한 time series model과 blend.

### 1.4 Updated-in-Place Tables
> 일부 table은 기존 record가 사후 변경(예: shipping date가 NULL→채워짐)되어 최신 date에 항상 anomaly가 있는 것처럼 보인다.

- table 유형: **static**(time column 없음, 매일 snapshot 필요), **log**(append만, created_at으로 partition), **updated-in-place**(created_at + updated_at, 주의 필요).
- row count 변화 시각화로 패턴 식별: **대각선**(신규 data가 며칠에 걸쳐 성숙, 폭=성숙 기간), **수평선**(특정일 batch로 historical 변경/migration), **수직선**(드묾).
- 해법: **매일 snapshot을 떠서 어제 관측한 어제 data와 비교** → updated-in-place로 인한 가짜 anomaly 배제. 어느 table이 그런지 알기 어려우니 항상 그렇다고 가정하고 추가 연산 감수. 단 snapshot 누적 전엔 warm start 어려움.

### 1.5 Column Correlations
> 대부분 dataset엔 강한 correlation 구조가 있어, 하나의 issue가 여러 column에 영향을 주고 중복 alert를 유발할 수 있다.

- correlation 원인: 같은 data의 다중 표현, identifier 계층, causal funnel. data가 "fan out"되는 pipeline에서 자주 발생(location ID 결측 → join 실패 → 모든 location metadata 결측).
- 해법: SHAP value로 **row-level correlation**을 보고 같은 row들에서 동시에 anomalous한 column들을 clustering해 **단일 issue**로 제시(예: Greek Yogurt 결측이 Aisle·Product·Brand column에 동시 출현).

---

## 2. Model Testing
> human label dataset은 너무 비싸고 주관적이므로, synthetic anomaly("chaos") 탐지를 real issue 탐지의 proxy로 사용한다.

- benchmark 알고리즘: 대표 dataset 수집 → chaos 주입 전/후로 model 실행 → runtime·accuracy 측정 → parameter·feature dampening으로 fine-tune → 반복. ("렌치를 피할 수 있으면 공도 피한다" - Dodgeball.)

### 2.1 Injecting Synthetic Issues
> Netflix Chaos Monkey처럼, SQL로 benchmark dataset을 조작해 실제 issue를 시뮬레이션한다.

- real issue는 일부만 영향 주므로 segment·random column·random 비율 등으로 다양하게 적용 후 sensitivity·specificity·AUC 등 측정.
- 예: numtickets max=30인데 최신 date의 venuestate='NY' record 30%만 40으로 변경(갑작스럽고 미묘한 anomaly). CASE WHEN random()<0.3 THEN 40 ... 식 SQL.
- Anomalo의 **Chaos Llama** 라이브러리 연산: ColumnGrow(랜덤 배수), ColumnModeDrop(mode 값 row 제거, 희귀 mode 방지 threshold), ColumnNull(일부 NULL화), ColumnRandom(범위 내 랜덤 대체), TableReplicate(원본에서 sampling해 row 추가).

### 2.2 Benchmarking
> benchmark는 여러 backtest로 구성되며, 각 backtest는 연속된 날짜의 historical sample을 순차적으로 model에 돌린다.

- chaos 없이 1회(baseline anomaly score·동적 threshold 기록), 이후 매일 random chaos 주입해 재실행하며 sensitivity 측정.
- 예시 backtest(Apr29~May28): chaos 없는 score는 초기 외엔 0 근처(예측 가능). threshold는 10에서 시작해 3일 유지 후 exponential decay하여 30일 후 0.3 미만(moderate anomaly 탐지 가능). chaos score는 초반 threshold가 높아 억제되나 4일차쯤 threshold 통과 — 30~90일 지나며 calibration 향상.
- 일부 날 alert 안 되는 이유: threshold가 아직 높음, chaos가 매우 희귀(1% 등), chaos가 실제로 data를 안 바꿈(이미 99% NULL인 column에 5% NULL 추가).

### 2.3 Analyzing performance
> AUC, F1, precision, recall을 dataset·실행 일수·chaos 유형·chaos fraction별로 slice해 분석한다.

- **chaos fraction**이 특히 유용: 전체 적용 시 거의 완벽해야 하고, 어느 비율부터 sample size상 탐지 불가해지는지 파악.
- **AUC**: score를 chaos 여부와 비교. 1% chaos에서 ~0.50, 50%+에서 ~0.80. 실제 성능은 더 좋음(negative에 진짜 anomaly 포함, chaos가 fraction보다 적게 영향, 30일만 측정하지만 90일까지 계속 개선).
- **precision**(chaos일 때 alert 비율, false positive): 소량 chaos 시 ~50%, 대량 시 ~90%. **recall**(탐지한 chaos 비율): 초기 0 근처→과반 chaos 시 ~50%. recall 개선은 threshold를 낮추거나 빨리 decay시키면 가능하나 false positive·alert fatigue 증가.
- **F1** = 2*(precision*recall)/(precision+recall). 실무에선 false positive/negative 비용을 직접 추정해 calibration 결정.

### 2.4 Pseudocode
> Ch4의 detect_anomalies를 활용해 calculate_anomaly_scores → backtest → benchmark → calculate_global_auc 흐름으로 벤치마킹한다.

```python
def backtest(table, time_column, start_date, number_of_days):
    for day in range(number_of_days):
        current_date = start_date + dt.timedelta(days=day)
        prior_date = current_date - dt.timedelta(days=1)
        anomaly_scores.append(calculate_anomaly_scores(table, time_column, current_date, prior_date))
        table_chaos = generate_random_chaos(table, time_column, current_date)
        chaos_anomaly_scores.append(calculate_anomaly_scores(table_chaos, time_column, current_date, prior_date))
    return anomaly_scores, chaos_anomaly_scores
# benchmark()로 여러 config 실행 후 roc_auc_score로 global AUC 산출
```

---

## 3. Improving the Model
> benchmark 통계는 model 변경이 올바른 방향인지 검증하는 데 가장 중요하게 쓰인다.

- 예: string column 패턴에 더 민감한 feature 추가 → runtime은 유의하게 증가(최적화 필요 신호)했으나 false negative 감소로 precision·recall 개선.
- AUC 개선은 절대값으론 작아 보여도 0.5 초과분 기준으로 보면 유의미: `(0.624-0.5)/(0.617-0.5) - 1 = 5%` 개선.

---

## Summary (핵심 정리)
- seasonality·time-based feature·chaotic table·updated-in-place·column correlation 각각에 mitigation이 있어야 model이 적절히 alert한다.
- 핵심 mitigation: 다중 lookback sampling, 시간상관 feature 제거, 동적 threshold(decay), 매일 snapshot 비교, SHAP correlation clustering.
- human label 대신 synthetic chaos 주입(Chaos Llama)으로 model을 테스트한다.
- backtest·benchmark로 AUC·precision·recall·F1을 chaos fraction 등으로 분석해 model을 반복 개선한다.
