# 05. Data Quality Metrics and Visualization

## 챕터 개요 (3줄 요약)

- DQS 적용으로 생성된 대량의 data quality metric(예제에서 550개)을 heatmap·3D bar chart로 시각화해 anomaly를 빠르게 식별한다.
- stop-light 방식(green=Valid, yellow=Suspect, red=Invalid)과 prioritization logic으로 datum별 summary metric을 산출한다.
- Data Management function이 anomaly를 단계적으로 remediation하면 data volume status가 IV → V로 전환되어 fit-for-purpose를 충족한다.

---

## 1. Data Quality Metrics

> Data quality metric은 DQS validation 적용 결과(V/S/IV)이며, anomaly는 metric이 suspect 또는 invalid일 때 식별된다.

- 예제 security master(25 records × 11 elements = 275 datum)에 completeness·timeliness·accuracy·precision·conformity DQS 적용 → 총 **550 metric** 생성 (Valid 441, Invalid 79, Suspect 30).
- 실무에선 data volume이 훨씬 커서 metric 수가 방대 → visualization이 anomaly 인식에 필수.

---

## 2. Data Quality Visualization

> 인간 뇌는 이미지를 텍스트보다 ~60,000배 빠르게 처리하므로 heatmap·3D chart로 품질을 빠르게 파악한다.

### Heatmap (stop-light)

- **green(V)**: valid tolerance 충족, fit for purpose.
- **yellow(S)**: out-of-tolerance 접근, 조사 필요.
- **red(IV)**: valid·suspect 모두 미충족, not fit for purpose.

### Prioritization logic (datum별 summary metric)

```text
if any metric == IV  -> summary = IV
elif any metric == S -> summary = S
else                 -> summary = V
```

### 단계적 remediation (IV → V)

- Data Management function이 data volume별로 순차 교정: Ticker/Exchange → Issue Name → (securities V) → Bid/Ask/Spread → (prices V) → Market Cap/Scale/PE → (fundamentals V) → Consensus Rec/Date → (consensus recs V) → holdings → (performance V).
- holdings가 valid가 되어야 performance 데이터를 정확히 생성 가능 (의존성).

### Bar chart

- 3D bar chart로도 표현 가능. **valid metric을 포함하면 suspect/invalid가 잘 안 보임** → suspect·invalid만 표시한 chart가 anomaly 식별에 더 유용.

---

## Summary (핵심 정리)

- Visualization은 manufacturing의 quality assurance(raw material 물성을 control spec tolerance와 비교)와 동일한 발상으로, datum dimension을 DQS tolerance와 대조한다.
- 목적은 investigation·remediation이 필요한 anomaly를 빠르게 식별하고, data volume이 valid tolerance를 충족해 fit-for-purpose임을 확인하는 것.
- 다음 챕터(6장)는 pre-use data validation(DQS framework)의 가치를 보여주는 operational efficiency cost model을 template로 제시한다.
