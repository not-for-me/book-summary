# 15. TikTok's Personalized Recommender: The World's Most Valuable AI System

## 챕터 개요 (3줄 요약)
- TikTok "For You" 피드를 모방한 real-time 개인화 video 추천 시스템을 retrieval-and-ranking 아키텍처로 설계·구축한다.
- two-tower embedding model(retrieval)과 XGBoost ranking model을 streaming/batch/vector-embedding feature pipeline으로 학습·서빙한다.
- agentic 자연어 video search를 추가하고, MLOps의 "dirty dozen" fallacy와 AI builder의 윤리적 책임으로 책을 마무리한다.

---

## 1. Introduction to Recommenders
> 추천은 content-based → i2i/u2i(collaborative filtering/factorization machine) → retrieval-and-ranking으로 진화.

- content-based(genre/director 등, content feature만, 확장 쉬움), interaction 기반 **i2i**("이걸 산 사람이...")·**u2i**(사용자 중심).
- collaborative filtering·factorization machine은 대용량·sparse·real-time에 한계.
- feature 그룹(Table 15-1): user profile, video, interactions(TBs/PBs), real-time context(streaming), in-session(ODT), graph/social.

---

## 2. Retrieval-and-Ranking Architecture
> ① retrieval(vector index로 수백 후보, high recall) ② ranking(precision, 지표 최적화 정렬).

- 시스템 과제:
  - **nonstationarity**: 선호·trending 초 단위 변화 → fresh feature(stream, TikTok은 Flink subsecond + Cassandra/Redis) + 잦은 재학습(Monolith는 분당, 본서는 시간당 batch).
  - **sparse feature**: 고cardinality categorical(one-hot 희소, cold-start, overfit) → **embedding**(고차원 sparse→저차원 dense). user·video 두 데이터를 **two-tower**로 interaction 데이터로 연결.
  - **retrieval**: 수십억 video에서 수백 후보를 ms 내(vector index ANN). user embedding으로 query.
  - **ranking**: 후보를 precision·utility로 정렬(XGBoost + real-time feature). YouTube 2012: view count→watch time 전환 효과.
  - **scalability**: 수백만 동시 요청·PB. Hopsworks vector index(OpenSearch, FAISS, <10ms), feature store(RonDB, 10-20ms p99 batch).
  - **data**: 공개 데이터 부족 → synthetic. interaction이 핵심.
- **interaction_score**: 0(미시청)/1(시청)/2(좋아요)/3(공유) + watch_time.

---

## 3. Real-Time Personalized Recommender
> MVPS: 4 feature group, 2 feature view, 3 model.

- 스택: Feldera(stream), Polars(batch), PySpark(vector embedding), TensorFlow Recommenders(two-tower), XGBoost(ranking), KServe/FastAPI, Hopsworks. (Netflix는 추천·search를 단일 retrieval-and-ranking 인프라로 통합.)

### Feature Pipelines
- interaction은 Kafka(event sourcing→lakehouse). 모든 feature group은 offline+online.
- **streaming**(Feldera, backfill 가능): video_stats_fg(cnt_views/ctr per h/d/w/m), user_activity_fg(recently_viewed, last_login, mean/std_session_duration).
- **vector embedding**(PySpark): video_attrs_fg(name/genre/rating + video_stats). static+dynamic(trending) feature. OpenSearch가 bottleneck(수만 updates/sec) → popularity threshold로 갱신 빈도↑. GPU는 ~10x embedding 처리량.
- **batch**: user_profile_fg(location/age/gender, slowly changing, daily). Great Expectations validation.

### Training Pipelines
- retrieval_fv·ranking_fv를 interactions(label=interaction_score)에서 join. training data를 CSV로 materialize.
- **two-tower model**: user tower(user feature)·video tower(video feature) 두 NN. 각 tower는 embedding layer(ID)·feedforward(numerical)·transformer(text/sequential)·CNN(image)로 길이 d embedding 출력.
  - similarity(dot product/cosine) → positive(1,2,3)/negative(0) 이진 → contrastive loss(InfoNCE/sampled softmax)로 양 tower weight 업데이트. fine-grained는 ranking이 처리.
  - **negative sampling**: interaction 없으면 random video로 bootstrap.
- **vector index 구축**: 학습된 video tower로 interaction 데이터에서 video embedding 생성·write. 조회 시 user embedding으로 top N(50-1000) ANN.
- **ranking model**: N 후보를 richer feature(user×video cross, real-time)로 rerank. Wide&Deep/DCN/DeepFM. 지표: NDCG, MRR, MAP@K.

### Online Inference Pipeline (KServe/FastAPI)
- 단계: ① **retrieval**(user_id로 user feature → user embedding → vector index 200 후보), ② **filtering**(video feature 조회, 연령 부적합 제외), ③ **ranking**(model.predict, 병렬 CPU).
- P95 45ms 목표: user feature 1ms + embedding 4ms, ANN 10ms, filter <1ms, video feature batch lookup 23ms, ranking 5ms, async 로깅 1ms, ODT <1ms. tail at scale 주의.
- 로깅으로 ranking 성능 모니터링(outcome=interaction 데이터, 수분 후). 저하 시 ranking 또는 embedding 재학습/재설계.

---

## 4. Agentic Search for Videos
> LLM agent로 자유 텍스트 video 검색.

- video 내 질문: active video_id로 video_tags 조회 → LLM이 적절 tag 선택 → pos_ms로 이동.
- 전체 video 질문: transcripts vector index ANN으로 유사 transcript 찾아 재생.
- video 전사·frame 추출·tag feature pipeline 추가.

---

## 5. The Dirty Dozen of Fallacies of MLOps
> production 실패를 부르는 12가지 잘못된 가정.

1. monolithic ML pipeline로 다 한다 → real-time/agentic 불가. **FTI로 분해**.
2. AI 데이터는 static → 실제는 dynamic, 반복 가치 생성.
3. 모든 변환은 동일 → MIT/MDT/ODT 구분, skew 방지.
4. feature store 불필요 → time-series point-in-time·real-time context·skew 방지에 필요.
5. experiment tracking이 MLOps 시작 → premature optimization, model registry로 시작.
6. MLOps = DevOps for ML → 데이터·모델 검증·drift 모니터링 추가 필요.
7. model versioning만으로 안전 → model deployment와 feature view 버전 결합 필요.
8. data versioning 불필요 → reproducible training data에 필요(late data·ingestion timestamp).
9. model signature = deployment API → 다름, deployment API 명시.
10. 예측 지연 = model.predict 시간 → 전처리(feature/RAG)·후처리(로깅) 포함.
11. LLMOps ≠ MLOps → GPU·scale 지원하면 LLMOps는 FTI 따르는 MLOps.
12. ML orchestrator 필수 → Flink/Feldera 등 못 돌림, lineage는 feature store/model registry가 source of truth.

---

## 6. Ethical Responsibilities of AI Builders
> 법 준수 너머 직간접 harm 방지 책임.

- 예: TikTok 추천이 13세로 추정 사용자에 1시간 후 depression/self-harm 영상만 노출(RTÉ 2024). 합법이나 비윤리적 시스템은 고치거나 떠나 whistleblow.
- Vasa 함선(1627): 불가능을 알면서 만든 엔지니어, 진수 시 침몰·약 30명 사망. harm 주는 AI를 만드는 개발자가 되지 말 것.

---

## Summary (핵심 정리)
- TikTok형 개인화 video 추천을 retrieval-and-ranking(two-tower retrieval + ranking model)로 구축했다.
- streaming·batch·vector-embedding feature pipeline, embedding·ranking training pipeline, retrieval·ranking online inference를 다뤘다.
- LLM 기반 agentic 자유 텍스트 video search를 추가했다.
- MLOps/LLMOps의 dirty dozen fallacy를 피하고, AI builder의 윤리적 책임을 강조하며 책을 마무리했다.
