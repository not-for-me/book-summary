# 11. Simple Entity Resolution in Graphs

## 챕터 개요 (3줄 요약)
- entity resolution(누가 누구인가/무엇이 무엇인가)을 정의하고, MovieLens와 Kaggle 두 데이터셋을 병합한 과정을 단계별로 공개한다.
- strong identifier(IMDB/TMDB ID) 분포를 파악해 매칭 알고리즘을 만들고, false positive 오류를 발견·해결한다.
- entity resolution 대부분은 graph가 불필요하며, strong identifier·edit distance 후 보조적으로만 graph structure가 유용함을 설명한다.

---

## 1. Entity Resolution 정의

> 서로 다른 데이터 소스에서 동일 entity를 식별하는 복잡한 문제.

- "Jon Smith == John Smith?", "Das Versprechen == The Promise?"
- 산업별 오차 허용도 다름(의료=치명적, 영화=UX 저하 정도).
- 보통 강한 고유 식별자가 없어 keys·values에서 논리적 identity를 계산.

---

## 2. Entity Resolution 프로세스 & "good enough"

> 모든 a,b에 대해 f(a,b) > 임계값 t면 a=b로 판정.

- 단계: 소스 식별 → key/value 분석 → strong/weak identifier 매핑 → "good enough"까지 반복(매칭→오류식별→해결→반복).
- 수학적 정의: 데이터 D의 모든 a,b에 f 적용, score > t면 동일.
- C360의 통합 graph는 entity resolution 결과물 — 문제 기술이 graph로 쉬워 graph를 오용하기 쉬움.

---

## 3. MovieLens 데이터 모델링 (6 파일)

> links → movies → ratings → tags → genome 순으로 점진적 schema 구축.

- **links.csv**: movieId/imdbId/tmdbId (strong identifier 소스, IMDB 100% / TMDB 98.95% 커버).
- **movies.csv**: title + genre → Movie vertex, Genre vertex, belongs_to edge.
- **ratings.csv**: 2천만 평점 → User vertex, rated edge(rating, timestamp).
- **tags.csv**: tagged edge. **genome**: Tag vertex(relevance score) + topic_tagged edge.
- 가장 중요한 strong identifier = movie_id (모든 파일이 movie 연결에 사용).

---

## 4. Kaggle 데이터 추가

> movie 상세 + 배우 정보로 모델 augment.

- **AllMoviesDetailsCleaned.csv**: 329,044 movie. id=tmdb_id, imdb_id. TMDB 100% / IMDB 76%(24% 누락). budget/revenue/popularity 등 6 property + genre 추가.
- **AllMoviesCastingRaw.csv**: 영화당 배우 5명 → Actor vertex, acted_in edge, collaborated_with edge(같은 영화 출연, year).
- 식별자 커버리지 불일치(Kaggle=TMDB 100%, MovieLens=IMDB 100%)가 매칭을 복잡하게.

---

## 5. 매칭 프로세스

> 항상 각 시스템의 strong identifier **분포** 파악부터 시작.

```
For each movie_k in Kaggle:
  movie_m = MATCH MovieLens by tmdb_id of movie_k
  if movie_m and imdb_id matches:
     이중 확인(imdb_id로도 조회) 후 UPSERT
  elif MATCH by imdb_id:
     UPSERT
  else:
     INSERT (MovieLens에 없음)
```

- 두 소스 모두에 있는 영화 26,853개 매칭. UPSERT 사용(Cassandra라 가장 빠름).
- 양쪽 ID를 모두 써야 함 — 어느 한쪽도 100% 신뢰 불가.

---

## 6. False Positive 해결

> entity resolution이 동일하지 않은 두 reference를 연결하는 오류.

- **MovieLens 내부 false positive**: 같은 TMDB ID가 다른 IMDB ID 가리킴(17건). 예: The Promise/Das Versprechen이 같은 tmdb_id 105045.
- 17쌍(34건) 제거, Kaggle을 ground truth로 사용.
- 병합 시 추가 143건(IMDB 일치, TMDB 불일치) 오류 — 조사 결과 모두 Kaggle이 정확.
- title 불일치 이유: (year) 64%, "The" 25%, "A" 3.5%, 실제 다름 6.3%, 언어 차이 0.7%.

---

## 7. Graph Structure의 역할

> 대부분의 entity resolution은 graph 불필요 — strong identifier·edit distance가 우선.

- 예: 같은 title·tmdb_id, 다른 imdb_id인 두 "The Warrior (2001)" → 데이터만으론 판별 불가.
- 배우 정보를 edge로 추가하면 공통 배우 0명 → 서로 다른 영화임을 즉시 확인.
- 순서: (1) strong identifier 정확 매칭 → (2) edit distance(이름) → (3) relationship이 의미 있으면 graph 도입.
- 최종 병합: 329,469 movie (양쪽 26,853 / IMDB 누락 78,480 / TMDB 누락 237).

---

## Summary (핵심 정리)
- entity resolution = 소스 간 동일 entity 식별. f(a,b)>t면 동일, "good enough"까지 반복.
- MovieLens(IMDB 100%)+Kaggle(TMDB 100%) 식별자 커버리지 불일치 → 양쪽 ID 모두 사용하는 매칭 필요.
- 항상 strong identifier 분포 파악부터 시작. UPSERT로 병합(Cassandra).
- false positive(다른 것을 같다고 연결): MovieLens 내부 17건 + 병합 143건, Kaggle을 ground truth로.
- entity resolution 대부분은 graph 불필요 — strong identifier→edit distance 후 보조적으로 relationship(공통 배우 등) 활용.
