# PostgreSQL 전문 검색(Full-Text Search) 완벽 가이드

- **키워드**: tsvector, tsquery, lexeme, tokenization, normalization, ts_rank, setweight, ts_headline, GIN index, GiST index
- **출처**: [Just Use Postgres! - Chapter 6: Postgres for full-text search](https://learning.oreilly.com/library/view/just-use-postgres/9781633435698/Text/chapter-6.html)
- **3줄 요약**:
  - PostgreSQL은 별도의 검색 엔진(Elasticsearch 등) 없이도 내장 전문 검색 기능으로 토큰화, 정규화, 검색, 랭킹, 하이라이팅을 모두 수행할 수 있다.
  - 원본 텍스트를 `tsvector` 타입의 lexeme으로 변환하여 저장하고, 검색 쿼리를 `tsquery`로 변환한 뒤 `@@` 연산자로 매칭한다.
  - GIN 인덱스와 GiST 인덱스를 활용하면 전문 검색 성능을 수백 배 향상시킬 수 있다.

---

## 1. 전문 검색 처리 파이프라인

MySQL의 `FULLTEXT` 인덱스와 개념적으로 유사하지만, PostgreSQL은 훨씬 세밀한 제어가 가능하다. 전체 처리 흐름은 다음과 같다.

```
+------------------+     +------------------+     +------------------+
|   Raw Document   | --> |  Tokenization    | --> |  Normalization   |
|  (TEXT column)   |     |  (parser splits  |     |  (dictionary     |
|                  |     |   into tokens)   |     |   stems/removes) |
+------------------+     +------------------+     +------------------+
                                                          |
                                                          v
+------------------+     +------------------+     +------------------+
|   Search Query   | --> | tsquery          | --> |  @@ Match        |
|  (user input)    |     | (normalized      |     |  (tsvector vs    |
|                  |     |  query form)     |     |   tsquery)       |
+------------------+     +------------------+     +------------------+
                                                          |
                                                          v
                                                  +------------------+
                                                  |  Rank & Highlight|
                                                  |  (ts_rank,       |
                                                  |   ts_headline)   |
                                                  +------------------+
```

**MySQL과의 핵심 차이점**: MySQL의 `MATCH ... AGAINST` 구문은 단일 함수 호출이지만, PostgreSQL은 tsvector 생성 → tsquery 변환 → 매칭의 각 단계를 개별 함수/연산자로 분리하여 제공한다. 이로 인해 가중치 부여, 언어별 설정, 고급 연산자 조합 등이 가능하다.

---

## 2. 토큰화(Tokenization)와 정규화(Normalization)

### 2.1 토큰화

PostgreSQL의 내장 파서(`pg_catalog.default`)가 원본 텍스트를 토큰으로 분리한다. `ts_debug` 함수로 과정을 확인할 수 있다.

```sql
SELECT token, description, lexemes, dictionary
FROM ts_debug('5 explorers are traveling to a distant galaxy');
```

| token | description | lexemes | dictionary |
|---|---|---|---|
| 5 | Unsigned integer | {5} | simple |
| explorers | Word, all ASCII | {explor} | english_stem |
| are | Word, all ASCII | {} | english_stem |
| traveling | Word, all ASCII | {travel} | english_stem |
| distant | Word, all ASCII | {distant} | english_stem |
| galaxy | Word, all ASCII | {galaxi} | english_stem |

### 2.2 정규화

정규화 과정에서 사전(dictionary)이 각 토큰을 lexeme으로 변환한다.

```
+---------------------+     +---------------------+
| Token: "explorers"  | --> | Lexeme: "explor"    |
| (english_stem dict) |     | (suffix -er, -s     |
|                     |     |  removed by stemmer) |
+---------------------+     +---------------------+

+---------------------+     +---------------------+
| Token: "are"        | --> | Lexeme: {}          |
| (english_stem dict) |     | (stop word removed) |
+---------------------+     +---------------------+

+---------------------+     +---------------------+
| Token: "5"          | --> | Lexeme: {5}         |
| (simple dict)       |     | (no change)         |
+---------------------+     +---------------------+
```

핵심 사전 두 가지:
- `simple` 사전: 언어 규칙 없이 토큰을 그대로 유지. 숫자, URL, 이메일 등에 사용.
- `english_stem` 사전: 영어 어간 추출(stemming) + 불용어(stop word) 제거 수행.

불용어는 완전히 삭제되지 않고 빈 lexeme `{}`로 매핑되어 **위치 정보가 보존**된다. 이는 `<->` (FOLLOWED BY) 연산자에서 중요하다.

### 2.3 검색 설정(Configuration)

현재 기본 설정 확인:
```sql
SHOW default_text_search_config;
-- pg_catalog.english
```

설정별 사전 매핑 확인:
```sql
\dF+ english
```

사용 가능한 모든 설정 목록:
```sql
\dF
```

PostgreSQL은 아랍어, 러시아어, 한국어 등 다양한 언어의 사전 설정을 기본 제공하며, 커스텀 설정도 생성 가능하다.

---

## 3. 데이터 준비: tsvector 생성 및 저장

### 3.1 to_tsvector 함수

원본 텍스트를 lexeme 리스트로 변환한다.

```sql
SELECT to_tsvector('The explorers must save the fragile peace between Earth and the aliens.');
-- 결과: 'alien':12 'earth':9 'explor':2 'fragil':6 'must':3 'peac':7 'save':4
```

각 lexeme 뒤의 숫자는 **원본 텍스트에서의 위치**를 나타낸다. 불용어 "The"가 제거되었지만 "explor"의 위치는 여전히 2이다(위치 정보 보존).

### 3.2 저장 전략 비교

```
+----------------------------+----------+--------+---------+
|         Strategy           |  Speed   | Space  | Flex    |
+----------------------------+----------+--------+---------+
| Generate on-the-fly        |  Slow    | None   | High    |
| (every query runs          |          |        |         |
|  to_tsvector)              |          |        |         |
+----------------------------+----------+--------+---------+
| Store in tsvector column   |  Fast    | Medium | High    |  <-- recommended
| (+ GIN/GiST index)        |          |        |         |
+----------------------------+----------+--------+---------+
| Index directly             |  Fast    | Low    | Low     |
| (no column, index only)   |          |        | (config |
|                            |          |        |  fixed) |
+----------------------------+----------+--------+---------+
```

### 3.3 Stored Generated Column으로 lexeme 저장

```sql
ALTER TABLE omdb.movies
ADD COLUMN lexemes tsvector
  GENERATED ALWAYS AS (
    to_tsvector('english',
      coalesce(name, '') || ' ' || coalesce(description, ''))
  ) STORED;
```

핵심 포인트:
- `GENERATED ALWAYS ... STORED`: 원본 데이터(name, description)가 변경되면 lexeme이 자동 재생성된다. MySQL의 generated column과 동일한 개념이다.
- `'english'`을 명시적으로 전달해야 한다. 생략하면 `default_text_search_config` 설정에 의존하게 되어 **immutable 조건 위반**으로 오류가 발생한다.
- `coalesce(column, '')`: NULL 값이 `to_tsvector`에 전달되면 결과도 NULL이 되므로 빈 문자열로 대체한다.

---

## 4. 전문 검색 실행: tsquery와 @@ 연산자

### 4.1 tsquery 변환 함수 비교

```
+-------------------------+--------------------+-----------------------------+
|       Function          |  Operator Between  |  Use Case                   |
|                         |  Lexemes           |                             |
+-------------------------+--------------------+-----------------------------+
| plainto_tsquery()       |  & (AND)           | Simple user input           |
+-------------------------+--------------------+-----------------------------+
| to_tsquery()            |  Manual            | Advanced: &, |, !, <->      |
|                         |  (user specifies)  | operators supported         |
+-------------------------+--------------------+-----------------------------+
| phraseto_tsquery()      |  <-> (FOLLOWED BY) | Exact phrase matching       |
+-------------------------+--------------------+-----------------------------+
| websearch_to_tsquery()  |  Mixed (auto)      | Raw user input with         |
|                         |                    | simple syntax (AND, OR, "") |
+-------------------------+--------------------+-----------------------------+
```

### 4.2 plainto_tsquery — 단순 검색

```sql
SELECT plainto_tsquery('a computer animated film');
-- 결과: 'comput' & 'anim' & 'film'
```

불용어 "a" 제거, 나머지 토큰 정규화, `&` 연산자로 결합. 모든 lexeme이 존재해야 매칭된다.

```sql
SELECT id, name FROM omdb.movies
WHERE lexemes @@ plainto_tsquery('a computer animated film');
```

### 4.3 to_tsquery — 고급 검색

`&`(AND), `|`(OR), `!`(NOT), `<->`(FOLLOWED BY) 연산자를 직접 조합할 수 있다.

```sql
-- AND + OR 조합
SELECT id, name FROM omdb.movies
WHERE lexemes @@ to_tsquery('computer & animated & (lion | clownfish | donkey)');

-- NOT 연산자 + 구문(phrase) 검색
SELECT id, name FROM omdb.movies
WHERE lexemes @@ to_tsquery('lion & !''The Lion King''');
-- 내부 변환: 'lion' & !( 'lion' <-> 'king' )
```

`<N>` 연산자로 lexeme 간 거리를 지정할 수 있다:
```sql
-- "return" 다음 3번째 토큰 위치에 "king"이 있는 문서
SELECT id, name FROM omdb.movies
WHERE lexemes @@ to_tsquery('return <3> king');
-- 결과: The Lord of the Rings: The Return of the King
```

---

## 5. 검색 결과 랭킹

### 5.1 ts_rank 함수

```sql
ts_rank([ weights float4[] ], vector tsvector, query tsquery [, normalization integer ])
```

lexeme의 출현 빈도와 위치를 기반으로 관련도 점수를 계산한다.

```sql
SELECT id, name, ts_rank(lexemes, to_tsquery('ghosts')) AS search_rank
FROM omdb.movies
WHERE lexemes @@ to_tsquery('ghosts')
ORDER BY search_rank DESC LIMIT 10;
```

`normalization` 파라미터: 0(기본값, 문서 길이 무시), 1(문서 길이로 나눔), 2(평균 빈도로 나눔). 비트마스크로 조합 가능.

### 5.2 setweight를 활용한 가중치 부여

제목(name)에 높은 가중치, 설명(description)에 낮은 가중치를 부여하여 제목 매칭에 우선순위를 줄 수 있다.

```
+---------+-------------------+
| Weight  |  Default Priority |
+---------+-------------------+
|   A     |   1.0 (highest)   |
|   B     |   0.4             |
|   C     |   0.2             |
|   D     |   0.1 (lowest)    |
+---------+-------------------+
```

stored generated column에 가중치를 적용하는 방법:

```sql
ALTER TABLE omdb.movies DROP COLUMN lexemes;
ALTER TABLE omdb.movies
ADD COLUMN lexemes tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'B')
  ) STORED;
```

결과 예시: `'ghost':1A,2B,15B` — 1A는 제목에서, 2B와 15B는 설명에서 출현했음을 의미한다.

tsquery에서도 가중치로 필터링 가능:
```sql
-- 제목(A 가중치)에 ghost가 포함된 영화만 검색
WHERE lexemes @@ to_tsquery('ghosts:A')
```

---

## 6. 검색 결과 하이라이팅

### ts_headline 함수

검색어에 매칭되는 텍스트 조각(fragment)을 추출하고 하이라이팅 태그를 삽입한다.

```sql
SELECT ts_headline(description, to_tsquery('pirates'),
  'MaxFragments=3, MinWords=5, MaxWords=10, FragmentDelimiter=<ft_end>')
  AS fragments
FROM omdb.movies
WHERE lexemes @@ to_tsquery('pirates:B')
ORDER BY ts_rank(lexemes, to_tsquery('pirates')) DESC LIMIT 1;
```

주요 옵션:

| 옵션 | 기본값 | 설명 |
|---|---|---|
| MaxFragments | 1 | 반환할 최대 조각 수 |
| MinWords | 15 | 조각의 최소 단어 수 |
| MaxWords | 35 | 조각의 최대 단어 수 |
| StartSel | `<b>` | 하이라이트 시작 태그 |
| StopSel | `</b>` | 하이라이트 종료 태그 |
| FragmentDelimiter | `...` | 조각 구분자 |

**주의**: `ts_headline`의 출력은 XSS 공격에 취약할 수 있다. 사용자 입력이 포함된 문서를 처리할 때는 반드시 HTML 새니타이징이 필요하다.

---

## 7. 인덱싱: GIN vs GiST

인덱스 없이 전문 검색을 수행하면 **Seq Scan**(전체 테이블 스캔)이 발생한다.

### 7.1 GIN (Generalized Inverted Index)

```
              +------------------+
              |   Root Page      |
              | "histor"|"peopl" |
              +--------+---------+
              /        |         \
             v         v          v
    +---------+  +---------+  +---------+
    | < histor|  |>=histor |  | >=peopl |
    |         |  |< peopl  |  |         |
    +---------+  +---------+  +---------+
        |            |   |         |
        v            v   v         v
    +-------+   +------+------+  +------+
    |"fight"|   |"glad"|"hist"|  |"save"|
    | tr1   |   | tr1  | tr1  |  | tr1  |
    | tr2   |   |      | tr2  |  | tr2  |
    | tr3   |   |      | tr3  |  |      |
    +-------+   +------+------+  +------+
```

각 고유 lexeme이 인덱스 엔트리가 되고, 해당 lexeme을 포함하는 모든 테이블 행을 참조한다. **역 인덱스(inverted index)** 구조이다.

```sql
CREATE INDEX idx_movie_lexemes_gin ON omdb.movies USING GIN (lexemes);
```

실행 계획 변화: `Seq Scan` 15.328ms → `Bitmap Index Scan` + `Bitmap Heap Scan` **0.150ms**

**제한**: GIN은 lexeme의 위치 정보나 가중치를 저장하지 않는다. 위치 기반 검색이 많다면 **RUM 인덱스**(확장)를 고려해야 한다.

### 7.2 GiST (Generalized Search Tree)

GiST는 각 문서를 **비트 시그니처**로 표현하여 트리 구조로 저장한다.

```
            Level 1 (Root)
        +----------+----------+
        | 11100111 | 00110101 |
        +-----+----+-----+----+
             /            \
       Level 2             Level 2
    +----------+        +----------+
    | 11000001 |        | 00100101 |
    | 10100110 |        | 00010001 |
    +-----+----+        +-----+----+
         /                    \
    Level 3 (Leaf)        Level 3 (Leaf)
    +----------+          +----------+
    | 11000001 | -> tr1   | 00100101 | -> tr3
    | 10100110 | -> tr2   | 00010001 | -> tr4
    +----------+          +----------+
```

시그니처 생성 원리: 각 lexeme에 고유 비트 패턴을 할당하고, 문서의 모든 lexeme 시그니처를 bitwise OR로 결합한다.

```sql
CREATE INDEX idx_movie_lexemes_gist ON omdb.movies USING GIST (lexemes);
```

기본 시그니처 길이: 124 bytes, 최대 2,024 bytes. 시그니처가 길수록 false positive이 줄어들지만 인덱스 크기가 커진다.

### 7.3 GIN vs GiST 비교

```
+------------------+------------------+------------------+
|    Criteria      |      GIN         |      GiST        |
+------------------+------------------+------------------+
| Lookup Speed     | Fast             | Slower           |
|                  | (exact lexeme    | (signature-based |
|                  |  matching)       |  + recheck)      |
+------------------+------------------+------------------+
| Build/Update     | Slower           | Faster           |
| Speed            | (complex struct) |                  |
+------------------+------------------+------------------+
| Index Size       | Larger           | Smaller          |
+------------------+------------------+------------------+
| False Positive   | None             | Possible         |
|                  |                  | (signature       |
|                  |                  |  collision)       |
+------------------+------------------+------------------+
| Best For         | Read-heavy       | Write-heavy      |
|                  | workloads        | workloads        |
+------------------+------------------+------------------+
```

**실무 가이드**: 대부분의 전문 검색 시나리오에서는 **GIN이 권장**된다. GiST는 인덱스 크기와 빌드/업데이트 속도가 중요한 경우에 선택한다.

---

## 8. 추가 확장 기능 참고

- **pg_trgm**: 트라이그램 기반 유사도 검색. 오타 보정, 자동완성 기능 구현에 활용. (`CREATE EXTENSION pg_trgm;`)
- **RUM index**: GIN과 유사하지만 lexeme의 위치 정보도 인덱스에 저장. `<->` 연산자를 자주 사용하는 경우 유리.
- **pg_search (ParadeDB)**: BM25 스코어링 알고리즘 등 Elasticsearch 수준의 고급 검색 기능 제공.

---

## Quiz

**Q1.** 다음 SQL의 실행 결과에서 "running" 토큰은 어떤 lexeme으로 변환되는가?
```sql
SELECT to_tsvector('english', 'The runners are running fast');
```
A) running  B) runner  C) run  D) runners

---

**Q2.** `to_tsvector` 함수를 stored generated column에서 사용할 때 `'english'` 설정을 명시적으로 전달해야 하는 이유는 무엇인가?

A) 한국어 문서를 처리하기 위해  
B) `default_text_search_config`는 세션 수준에서 변경될 수 있어 immutable 조건을 충족하지 못하기 때문  
C) english_stem 사전이 기본으로 설치되지 않기 때문  
D) PostgreSQL이 자동으로 언어를 감지하지 못하기 때문

---

**Q3.** 다음 쿼리에서 `<3>` 연산자의 의미를 설명하시오.
```sql
SELECT * FROM movies WHERE lexemes @@ to_tsquery('return <3> king');
```

---

**Q4.** `setweight` 함수로 제목에 'A' 가중치, 설명에 'B' 가중치를 부여한 후, 다음 tsquery가 검색하는 범위는 어디인가?
```sql
WHERE lexemes @@ to_tsquery('ghost:A')
```
A) 설명(description)에서만 검색  
B) 제목(name)에서만 검색  
C) 제목과 설명 모두에서 검색  
D) 가중치 A가 가장 높으므로 모든 문서를 반환

---

**Q5.** GIN 인덱스와 GiST 인덱스의 차이점으로 **틀린** 것은?

A) GIN은 각 lexeme을 개별 엔트리로 저장하고, GiST는 문서를 비트 시그니처로 저장한다  
B) GiST는 시그니처 충돌로 인해 false positive이 발생할 수 있다  
C) GIN은 lexeme의 위치 정보와 가중치를 인덱스에 저장한다  
D) 읽기 위주 워크로드에서는 일반적으로 GIN이 더 빠르다

---

**Q6.** `ts_headline` 함수를 사용할 때 보안상 주의해야 할 점은 무엇이며, 이를 방지하기 위한 방법은?

---

### Quiz 정답

**Q1.** C) `run` — `english_stem` 사전이 "running"을 어간 "run"으로 변환한다.

**Q2.** B) — Generated column의 표현식은 반드시 immutable이어야 한다. `default_text_search_config`는 세션/데이터베이스 수준에서 변경 가능하므로, 설정을 명시하지 않으면 immutable 조건을 위반하여 오류가 발생한다.

**Q3.** `<3>` 연산자는 "return" lexeme 이후 정확히 3번째 토큰 위치에 "king" lexeme이 존재해야 매칭됨을 의미한다. 예: "return of the king"에서 return(1) → of(2) → the(3) → king이 3번째 위치이므로 매칭된다.

**Q4.** B) — `:A`는 가중치 A가 부여된 lexeme에서만 검색하겠다는 의미이다. 제목에 A 가중치를 부여했으므로 제목에서만 검색한다.

**Q5.** C) — GIN은 lexeme 자체만 저장하며, 위치 정보나 가중치는 저장하지 **않는다**. 위치 정보가 필요하면 테이블 행을 다시 조회하거나 RUM 인덱스를 사용해야 한다.

**Q6.** `ts_headline`의 출력에는 원본 텍스트가 포함되므로, 사용자 입력이나 HTML 마크업이 포함된 문서를 처리할 경우 **XSS(Cross-Site Scripting) 공격**에 취약할 수 있다. 방지 방법: 입력 문서에서 HTML 태그를 제거하거나, `ts_headline` 출력 결과를 HTML sanitizer로 새니타이징한 후 웹 페이지에 렌더링해야 한다.