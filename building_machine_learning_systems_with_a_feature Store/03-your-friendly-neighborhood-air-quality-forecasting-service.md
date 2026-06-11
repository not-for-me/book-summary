# 03. Your Friendly Neighborhood Air Quality Forecasting Service

## 챕터 개요 (3줄 요약)
- 거주 지역 air quality sensor의 PM2.5를 weather forecast feature로 예측하는 첫 batch AI system을 MVPS 프로세스로 구축한다.
- feature backfill·daily feature pipeline·XGBoost training·batch inference의 4개 Python notebook + GitHub Actions/Pages로 무료 serverless 운영한다.
- Whisper + fine-tuned Llama 3 8B function calling으로 voice-driven UI를 더해 vector DB 없는 RAG를 구현한다.

---

## 1. AI System Overview
> prediction service card로 data source·prediction problem·UI·monitoring을 요약해 시스템을 정의한다.

- prediction problem: 선택한 sensor 위치의 향후 7일 PM2.5를 **regression**으로 예측.
- PM2.5는 wind speed/direction, temperature, precipitation과 상관 → weather forecast를 feature로 사용.
- data source: aqicn.org(air quality), open-meteo.com(weather). UI: web page + LLM UI. monitoring: hindcast graph.
- 스택: GitHub Actions/Pages, Hopsworks, Pandas, XGBoost, REST API, Streamlit + Whisper.
- (참고) Microsoft Aurora 같은 deep learning보다 좋은 데이터 + decision tree가 특정 지점에서 더 정확할 수 있음.

---

## 2. Data & EDA
> 데이터의 6가지 품질 차원(validity, accuracy, consistency, uniqueness, update frequency, completeness)을 점검한다.

- air quality: aqicn.org에서 historical CSV 수동 다운로드(API 없음) + API key로 실시간. `pm25`, `date` 컬럼 필요.
- 결측 점검: `df.isna().sum()`, `df.dropna()`.
- weather: Open-Meteo(historical/forecast 2개 API, key 불필요). precipitation, wind speed/direction, temperature 사용. geopy로 위경도 해석.
- 주의: forecast는 hourly, historical은 daily aggregate → 불일치하나 MVPS엔 충분.

---

## 3. Creating & Backfilling Feature Groups
> air_quality·weather 두 feature group을 만들고 Great Expectations로 data validation을 선언적으로 부착한다.

- `get_or_create_feature_group()`(idempotent)으로 생성, `event_time="date"`, primary_key 지정.
- **expectation_suite**(Great Expectations): 예) pm25가 0~500 범위인지 검증, 매 insert 시 실행. 실패 시 Slack/email 알림 + ingest-and-warn 또는 fail 정책.

---

## 4. Feature / Training / Inference Pipelines

### Feature pipeline
> backfill과 분리된 daily pipeline으로 오늘 PM2.5·weather·7일 forecast를 feature group에 insert.

- feature engineering 없음(numerical raw 그대로). 공통 함수는 mlfs/airquality 패키지.

### Training pipeline
> feature view로 feature 선택·join 후 XGBoost regressor를 학습해 model registry에 저장.

- **feature view**: 모델의 input feature + label schema 정의. `air_quality_fg.select(['pm25']).join(weather_fg.select_all(on=['city']))`.
- `train_test_split(test_size=0.2)`(random split, feature가 time-independent라서). `XGBRegressor().fit()`.
- 평가: MSE, R². feature importance PNG 저장. model registry 등록 시 auto-increment 버전.

### Batch inference pipeline
> model을 다운로드해 7일 weather forecast로 예측하고 Plotly chart PNG를 GitHub Pages에 publish.

- `days_before_forecast_day`로 예측 시점별 성능 평가. monitoring feature group에 예측·feature 저장.
- hindcast PNG: 예측값 vs 실측값 비교.

---

## 5. Running & Scheduling
> 로컬에서 notebook 검증 후 GitHub Actions로 daily 스케줄, GitHub Pages로 dashboard 게시.

- GitHub Actions(CI/CD): 무료 2000분/월. `nbconvert`로 notebook→Python 실행, `HOPSWORKS_API_KEY`는 repo secret. `cron`으로 매일 실행.
- GitHub Pages(무료, 도메인 제공)에 PNG 게시(`git-auto-commit-action`). Settings→Pages에서 활성화.
- 대안 orchestrator: Modal, Cloud Run, Step Functions, Airflow, Dagster, Mage AI.

---

## 6. Function Calling with LLMs
> Whisper로 음성→텍스트, fine-tuned Llama 3 8B로 텍스트→function call을 수행해 vector DB 없는 RAG를 구현한다.

- 흐름: Whisper 전사 → Llama 3 8B가 4개 함수 중 하나 + 파라미터를 JSON으로 반환 → 함수 실행(historical/forecast 조회) → 결과를 다시 LLM 프롬프트에 넣어 사람이 이해할 답변 생성.
- 함수: get_future_data_for_date, get_future_data_in_date_range, get_historical_air_quality_for_date, get_historical_data_in_date_range.
- LLM은 함수 declaration(이름·description·인자 설명)으로 용도 파악하나 직접 호출하지 않음 → 응답을 parse해 실행.
- GPU 필요(Colab T4 무료). Llama 3 8B를 4-bit quantize해 16GB GPU에서 구동(성능 영향 미미). Streamlit으로 UI 래핑.

---

## Summary (핵심 정리)
- 첫 AI system인 air quality 예측 서비스를 5개 Python 프로그램으로 분해해 구축했다.
- backfill, daily feature pipeline, on-demand training, batch inference(forecast/hindcast PNG), voice-driven LLM UI로 구성했다.
- GitHub Action YAML로 feature·inference pipeline을 daily 스케줄링했다.
- 개선 연습: lagged PM2.5 feature 추가, historical PM2.5 사용 시 risk 분석.
