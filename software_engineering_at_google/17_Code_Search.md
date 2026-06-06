# 17. Code Search

## 챕터 개요 (3줄 요약)
- Code Search는 거대 코드베이스를 읽고·이해하고·탐색하기 위한 웹 도구로, "코드에 대한 다음 질문을 한 번의 클릭으로 답한다"는 원칙으로 발전했다.
- IDE와 달리 편집이 아닌 브라우징·이해에 최적화되어 있고, 클라우드 백엔드로 검색·교차참조(Kythe)를 제공하며 다른 개발 도구의 표준 링크 플랫폼이 된다.
- AI 시대에 코드 이해가 개발·유지의 핵심일수록, 코드를 빠르게 검색·탐색·이해하게 만드는 인프라 투자는 측정 어렵지만 실질적인 생산성 레버리지다.

---

## 1. The Code Search UI (UI)
> 검색창은 핵심 요소로, 웹 검색처럼 제안(suggestion)과 즉각적 "find in files"(grep) + 관련도 랭킹 + 코드 특화 기능을 제공한다.

- 토큰 클릭으로 함수 정의·임포트 파일·버그 ID로 이동(Kythe 컴파일러 기반 인덱싱).
- Piper 통합으로 파일 이력·이전 버전·blame·삭제된 파일까지 조회.
- CLI·RPC API로도 제공 — 후처리나 대용량 결과에 유용.

---

## 2. How Googlers Use Code Search? (사용 패턴)
> 엔지니어의 작업은 "코드에 대한 질문에 답하기"로 볼 수 있다.

- Where(~16%): 정의·사용처·파일 위치 — 랭킹+풍부한 쿼리 언어; 결과 링크 공유가 코드 참조의 정규 방식.
- What(~25%): 파일 브라우징 — 변경 전 코드 이해(호출 계층·관련 파일 간 이동).
- How(~33%, 최다): 남들이 어떻게 했는지 예시 — 검색+교차참조 브라우징.
- Why(~16%): 코드가 왜 그렇게 동작하는가 — 디버깅; 특정 시점 코드 상태 탐색.
- Who/When(~8%): 누가/언제 도입했는가 — VCS 연동(blame·코드리뷰 이동).

---

## 3. Why a Separate Web Tool? (왜 별도 웹 도구인가)
> 로컬 IDE로는 안 되는 이유가 있다.

- Scale: Google 코드베이스가 단일 머신에 안 맞음 — 중앙 인덱스를 한 번 구축해 모두가 혜택(증분 인덱싱은 선형 비용); IDE식 개발자별 인덱싱은 사실상 제곱 확장.
- Zero Setup Global Code View: 셋업 없이 전체 코드베이스 즉시 탐색 — 재사용 라이브러리·예시 발견 용이.
- Specialization: IDE가 아니라는 점이 장점 — 모든 클릭이 의미(정의 이동/사용처)를 가짐; 흔히 IDE와 별도 탭으로 병행.
- 도구 통합: 로그/스택프레임/컴파일 에러/문서가 Code Search 링크를 정규 위치로 사용; 검색·교차참조·하이라이팅 API를 다른 도구(vim/emacs/IntelliJ 플러그인)에 노출.

---

## 4. Impact of Scale on Design (규모의 영향)
> 가장 큰 도전은 코퍼스 크기다.

- Search Query Latency: 빠른 UI는 절약되는 엔지니어 시간으로 정당화 — 200ms 이하면 반응성, 1초면 주의 이탈, 10초면 컨텍스트 전환(생산성 손실). 자주 쓰는 연산은 sub-200ms 목표.
- Index Latency: 작성·리뷰한 코드는 즉시 인덱싱 기대(검색 인덱스 ~10초 내). 단, 교차참조 인덱스는 증분 불가(변경이 전체에 영향) — 하루 1회 구축, 검색/교차참조 인덱스 간 불일치가 드문 이슈.

---

## 5. Google's Implementation (구현)
> 약 1.5TB 콘텐츠 인덱싱, 초당 ~200 쿼리, 중앙값 검색 지연 <50ms, 인덱싱 지연 <10초.

- 인덱스 진화: trigram → suffix array → sparse n-gram(brute-force 대비 500배+ 효율, 정규식도 빠르게).
- 표준 Google 검색 스택 활용(역인덱스·인코딩·즉시 인덱싱) — 구현 단순성 vs 성능 트레이드오프; 인덱스를 메모리→플래시로 이동.
- Ranking: 코드베이스가 클수록 중요 — query-independent 신호(파일 조회수, 참조수=page rank 유사, Kythe로 참조 추출)와 query-dependent 신호(토큰 매치·심볼/파일명 부스트·대소문자).
- Retrieval: 적은 고관련 파일을 다수 중에서 — supplemental retrieval(정의/파일명으로 쿼리 재작성); 결과 다양성(Java/Python 등 카테고리 균형).

---

## 6. Selected Trade-offs (트레이드오프)
> Code Search 구현은 다양한 절충을 수반한다.

```
Trade-off dimensions
Completeness:
  - drop non-text/generated/huge files (but devs must TRUST results)
  - all results vs top-ranked (shards ordered by priority; can fetch all)
  - head vs branches vs full history vs workspaces (Google indexes Piper history)
Expressiveness:
  Token  -> Substring -> Regex
  (more power, more index cost; trigram index -> regex via substring searches)
```

- 완전성: 일부 파일을 인덱스에서 빼면 자원 절약되나 신뢰 상실 위험 → 높은 한도로 과도 인덱싱 쪽으로 기움.
- 모든 결과 vs 최상위: 리팩토링·도구는 전체 결과 필요 — 샤드를 우선순위로 정렬해 두 use case 모두 지원.
- 이력 인덱싱: 단일 스냅샷보다 복잡·고비용이나, 삭제 코드 탐색·workspace 검색·시점별 탐색 가능(custom 압축으로 자원 2.5배).
- 표현력: 정규식은 강력하나 인덱싱 어려움; 토큰 인덱스는 작지만 코드 특수문자/식별자(CamelCase)에 부적합 → trigram 기반 substring 인덱스가 좋은 절충.

---

## Summary (핵심 정리)
- 개발자의 코드 이해를 돕는 것이 큰 생산성 향상이며, Google의 핵심 도구가 Code Search다.
- Code Search는 다른 도구의 기반이자 모든 문서·도구가 링크하는 중앙 표준 위치로서 부가 가치를 가진다.
- 거대 코드베이스가 별도 도구를 필요케 했고, 빠른 검색·브라우징·인덱싱(낮은 지연)과 모든 코드·결과를 신뢰성 있게 제공하는 것이 핵심 — AI 시대 코드 이해의 시니어 레버리지.
