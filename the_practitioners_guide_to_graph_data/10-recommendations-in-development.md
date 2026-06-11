# 10. Recommendations in Development

## 챕터 개요 (3줄 요약)
- Netflix Prize에서 비롯된 graph 기반 추천을 소개하고, collaborative filtering(특히 item-based)을 정의·분류한다.
- MovieLens+Kaggle 병합 데이터로 5 vertex/6 edge label schema를 구축하고 neighborhood/tree/path 쿼리를 복습한다.
- path counting / NPS / normalized NPS 세 가지 ranking model로 item-based collaborative filtering을 Gremlin으로 구현(실시간 확장성 문제는 Ch12 복선).

---

## 1. 추천 시스템 예제 (graph의 깊이)

> 추천은 graph를 neighborhood 단위로 걷는 것.

- 의료: 추천 시 맥락(1st neighborhood) 고려.
- 소셜미디어(LinkedIn "People you may know"): 친구의 친구 = 2nd neighborhood.
- 이커머스("similar products"): 더 깊은 neighborhood walk.

---

## 2. Collaborative Filtering 분류

> 개인 관심 + 커뮤니티 선호를 결합해 콘텐츠를 예측(filtering)하는 추천 방식.

- 추천 4분류: content-based(개인만), social data mining(커뮤니티만), **collaborative filtering**(둘 결합), hybrid.
- **user-based**: 비슷한 user 찾아 추천. **item-based**: 비슷한 item 찾아 추천.
- item-based가 graph 추천에 가장 인기(Amazon 1998 발명). 이 책은 item-based 사용.

---

## 3. Item-Based CF 과정 & 3 ranking model

> input(최근 평가 item) → method(평점 패턴으로 유사 item) → recommend(scoring model로 ranking).

- **Path counting**: 5-star 평점 경로 수 세기. 단순.
- **NPS**: like(≥4) - dislike(<4). 인기 영화에 수렴.
- **Normalized NPS**: NPS / 총 평점 수(degree). offbeat 영화에 다양성 부여(50/100=0.5 vs 20/25=0.8).

---

## 4. Movie Data Schema (MovieLens+Kaggle)

> 5 vertex(Movie/User/Genre/Actor/Tag) + 6 edge label.

```
single edge(1개): acted_in, belongs_to, topic_tagged
multi edge(CK, 다수): rated(timestamp,rating), tagged, collaborated_with(year)
```

- Avatar 매출 때문에 budget/revenue는 Int→Bigint.
- user_name은 Faker로 무작위 생성(MovieLens 실제 user와 무관).
- dsbulk로 vertex 5개/edge 6개 파일 로딩(rated 100k 샘플 등).

---

## 5. 쿼리 복습 — Neighborhood

> 1st neighborhood walk로 user의 영화 평점 조회 + group으로 liked/neutral/disliked 분류.

```
dev.V().has("User","user_id",134558).outE("rated").
  group().
  by(values("rating").coalesce(__.is(gte(4.5)).constant("liked"),
    __.is(gte(3.0)).constant("neutral"), constant("disliked"))).
  by(inV().values("movie_title").fold())
```

- coalesce = if/elif/else, choose()로 대체 가능.

---

## 6. 쿼리 복습 — Tree & Path

> collaborated_with edge로 actor "family tree"와 actor 간 shortest path.

```
// Tree: Kevin Bacon 협업자 3단계 (2009+)
dev.V().has("Actor","actor_name","Kevin Bacon").
  repeat(outE("collaborated_with").has("year",gte(2009)).inV().simplePath()).
  times(3).path().by("actor_name").by("year")

// Path: Kevin Bacon → Morgan Freeman shortest 3
repeat(...).until(has(...,"Morgan Freeman")).limit(3)  // BFS라 shortest
```

---

## 7. Model 1 — Path Counting (Gremlin)

> 최근 평가 영화 → 5-star 준 user → 그 user의 5-star 영화 group/count.

```
dev.V().has("User","user_id",694).
  outE("rated").order().by("timestamp",desc).limit(1).inV().
  aggregate("originalMovie").
  inE("rated").has("rating",gt(4.5)).outV().
  outE("rated").has("rating",gt(4.5)).inV().
  where(without("originalMovie")).
  group().by("movie_title").by(count()).unfold().order().by(values,desc)
```

---

## 8. Model 2 & 3 — NPS / Normalized NPS

> sack()으로 like(+1)/dislike(-1) 누적해 NPS, 그것을 movie degree로 나눠 normalize.

```
// Model 2 NPS: choose로 +1/-1
choose(values("rating").is(gte(4.0)),
  sack(sum).by(constant(1.0)), sack(minus).by(constant(1.0)))
...group().by("movie_title").by(sack().sum())

// Model 3 Normalized: project + math로 NPS/degree
by(project("numerator","denominator").
  by(sack().sum()).by(inE("rated").count()).
  math("numerator/denominator"))
```

- Model 3는 inE("rated").count()가 vertex마다 full partition scan → 매우 비쌈(실시간 한계, Ch12).

---

## Summary (핵심 정리)
- graph 추천은 Netflix Prize에서 부상 — neighborhood walk로 설명이 직관적.
- collaborative filtering = 개인+커뮤니티 선호 결합. item-based(유사 item)가 graph 추천의 주류.
- 3 ranking model: path counting(5-star 수), NPS(like-dislike), normalized NPS(NPS/degree, 다양성).
- schema: Movie/User/Genre/Actor/Tag + rated/tagged/acted_in/belongs_to/topic_tagged/collaborated_with.
- Gremlin: aggregate("originalMovie")+where(without), sack()으로 NPS, choose()로 +1/-1, project+math로 normalize. degree count는 비싸 실시간 불가(Ch12).
