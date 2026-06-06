# PostgreSQL 인덱스 완전 정복 — "Just Use Postgres!" Chapter 4

- 키워드: B-tree, Hash Index, Composite Index, Covering Index, Partial Index, Functional/Expression Index, EXPLAIN, Index Only Scan, Bitmap Index Scan, Seq Scan
- 출처: [Just Use Postgres! — Chapter 4: Indexes (O'Reilly)](https://learning.oreilly.com/library/view/just-use-postgres/9781633435698/Text/chapter-4.html)
- 3줄 요약
  1. PostgreSQL 인덱스는 **무엇을 인덱싱할 것인가**(scope/purpose)와 **어떻게 인덱싱할 것인가**(data structure/access method) 두 축으로 분류된다.
  2. EXPLAIN / EXPLAIN ANALYZE를 통해 실행 계획을 확인한 뒤 인덱스 생성 여부를 판단해야 하며, 과도한 인덱싱(over-indexing)은 쓰기 성능을 저하시킨다.
  3. Single-column, Composite, Covering, Partial, Functional 인덱스를 상황에 맞게 조합하면 테이블 스캔 없이 O(log N) 또는 O(1) 복잡도로 데이터를 조회할 수 있다.

---

## 1. 인덱스가 필요한 이유

인덱스 없이 조회하면 Sequential Scan(Seq Scan)이 발생하며, 알고리즘 복잡도는 **O(N)** 이다. 레코드가 100건이면 최대 100번, 1,000만 건이면 최대 1,000만 번 탐색해야 한다.

B-tree 인덱스를 사용하면 복잡도가 **O(log_b N)** 으로 감소한다. 분기 계수(branching factor) b=10 기준으로 데이터 양 대비 필요한 룩업 횟수는 다음과 같다.

| Table Size | Index Lookups (b=10) |
|---|---|
| 100 | 2 |
| 1,000 | 3 |
| 1,000,000 | 6 |
| 10,000,000 | 7 |
| 1,000,000,000 | 9 |

### B-tree 구조

```
                    +--------+
                    | Root   |
                    |  [10]  |
                    +---+----+
                   /         \
          +--------+        +--------+
          |Branch 1|        |Branch 2|
          | [6][10]|        |[15][20]|
          +--+--+--+        +--------+
            /    \
   +----------+ +----------+
   | Leaf 1   | | Leaf 2   |
   |[1][4][5] | |[7][8][9] |
   |  [6]     | |  [10]    |
   +----+-----+ +----------+
        |
    (linked list)
```

핵심 특성은 다음과 같다.

Root → Branch → Leaf 순으로 탐색하며, Leaf 노드에 실제 테이블 행에 대한 참조(table reference)가 저장된다. Leaf 노드끼리는 Linked List로 연결되어 있어 범위 검색(Range Query)과 정렬된 데이터 반환에 유리하다. 실무에서 B-tree의 분기 계수는 수백~수천에 달하므로 실제 필요한 노드 탐색 횟수는 매우 적다.

---

## 2. PostgreSQL 인덱스 분류 체계

PostgreSQL 인덱스는 두 가지 축으로 분류된다.

```
+-----------------------------------------------+
|          PostgreSQL Index Taxonomy             |
+----------------------+------------------------+
| WHAT we index        | HOW we index           |
| (Scope / Purpose)    | (Data Structure)       |
+----------------------+------------------------+
| Single-column        | B-tree  (default)      |
| Composite            | Hash                   |
| Covering (INCLUDE)   | GIN                    |
| Partial (WHERE)      | GiST / SP-GiST        |
| Functional/Expression| BRIN                   |
|                      | HNSW / IVFFlat         |
|                      | Bloom / RUM            |
+----------------------+------------------------+
```

`CREATE INDEX` 구문에서 **ON** 절이 "무엇을" 결정하고, **USING** 절이 "어떻게"를 결정한다. USING을 생략하면 기본값은 **B-tree**이다.

```sql
-- WHAT: single-column on columnA / HOW: default B-tree
CREATE INDEX idx_name ON tableA(columnA);

-- WHAT: single-column on columnA / HOW: GIN
CREATE INDEX idx_name ON tableA USING GIN (columnA);
```

MySQL 사용자를 위한 비교: MySQL InnoDB는 Clustered Index(PK) 기반이고 Secondary Index의 Leaf에 PK 값을 저장한다. PostgreSQL은 Heap Table 구조를 사용하며 모든 인덱스가 동등하게 테이블 행의 물리적 위치(ctid)를 참조한다.

---

## 3. EXPLAIN 문 활용법

### 기본 사용법

```sql
-- 실행 계획만 확인 (쿼리 실행 안 함)
EXPLAIN SELECT ... ;

-- 실행 계획 + 실제 실행 시간
EXPLAIN ANALYZE SELECT ... ;

-- 옵션 조합 (analyze + buffers + costs off)
EXPLAIN (analyze, buffers on, costs off) SELECT ... ;
```

`EXPLAIN ANALYZE`는 쿼리를 **실제로 실행**한다. UPDATE/DELETE에 사용할 경우 데이터가 변경되므로 주의해야 한다.

### 실행 계획에서 확인해야 할 핵심 지표

| 지표 | 의미 |
|---|---|
| Seq Scan | 전체 테이블 순차 스캔 — 인덱스 미사용 |
| Index Scan | 인덱스를 탐색 후 테이블에서 나머지 컬럼 조회 |
| Index Only Scan | 인덱스만으로 결과 반환 (테이블 접근 없음) |
| Bitmap Index Scan → Bitmap Heap Scan | 인덱스로 비트맵 생성 후 테이블 일괄 조회 |
| Heap Fetches: 0 | 테이블 접근 횟수 0 — 최적 상태 |
| Buffers: shared hit=N | 공유 버퍼(메모리)에서 읽은 페이지 수 |
| Buffers: shared read=N | 디스크에서 읽은 페이지 수 |
| Planning Time / Execution Time | 계획 수립 시간과 실행 시간 |

### Access Method 흐름도

```
+------------------+
| Query Received   |
+--------+---------+
         |
         v
+------------------+     No index      +------------------+
| Planner checks   | ----------------> | Seq Scan         |
| available indexes|                   | O(N) full scan   |
+--------+---------+                   +------------------+
         |
         | index exists
         v
+---+----------+----+---------------------+
|                   |                      |
v                   v                      v
+-------------+ +----------------+ +-------------------+
| Index Scan  | | Index Only     | | Bitmap Index Scan |
| (index +    | | Scan           | | + Bitmap Heap     |
|  table)     | | (index only)   | | Scan (bulk mode)  |
+-------------+ +----------------+ +-------------------+
```

**Bitmap Index Scan** 상세: 인덱스를 먼저 스캔하여 조건을 만족하는 행의 위치를 비트맵에 기록한 뒤, Bitmap Heap Scan 단계에서 테이블을 한 번에 일괄 접근한다. `Heap Blocks: exact=X`는 비트맵이 행 단위 위치를 정확히 기록했음을 의미하고, `Heap Blocks: lossy=Y`는 메모리 제한으로 페이지 단위로만 기록하여 각 페이지 내에서 재검증이 필요함을 의미한다.

---

## 4. Single-column 인덱스

### 4.1 B-tree (기본)

```sql
CREATE INDEX idx_score ON game.player_stats(score DESC);
-- USING btree 생략 시 자동으로 B-tree 적용
```

B-tree는 등호(=), 범위 비교(>, >=, <, <=, BETWEEN), ORDER BY, LIMIT에 모두 활용 가능하다. PostgreSQL은 PRIMARY KEY, UNIQUE 제약조건을 생성하면 자동으로 B-tree 인덱스를 만든다.

인덱스 생성 후에는 `ANALYZE` 명령을 실행하여 통계를 갱신해야 플래너가 새 인덱스를 즉시 활용할 수 있다.

```sql
CREATE INDEX idx_score ON game.player_stats(score DESC);
ANALYZE game.player_stats;
```

### 4.2 Hash 인덱스

```sql
CREATE INDEX idx_champion_title
  ON game.player_stats USING hash(champion_title);
```

Hash 인덱스는 등호(=)와 IN 연산만 지원하며, 범위 검색은 불가하다. 탐색 복잡도는 **O(1)** 이다. 단일 컬럼에서만 사용할 수 있고 Composite Index에는 적용할 수 없다.

MySQL과의 차이: MySQL InnoDB의 Adaptive Hash Index는 내부적으로 자동 생성되며 사용자가 직접 생성하지 못한다. PostgreSQL은 `USING hash`로 명시적으로 생성하며, PostgreSQL 10 이후부터 WAL 지원으로 Crash-safe하다.

---

## 5. Composite 인덱스 (다중 컬럼)

```sql
CREATE INDEX idx_region_score_win
  ON game.player_stats (region, score DESC, win_count DESC);
```

### 핵심 규칙

**컬럼 순서가 곧 정렬 순서이다.** 인덱스는 첫 번째 컬럼(leading column)으로 먼저 정렬되고, 동일 값 내에서 두 번째, 세 번째 컬럼 순으로 정렬된다.

```
Index: (region, score DESC, win_count DESC)

  region='APAC'
    score=10000, win=482
    score=9999,  win=300
    ...
  region='EMEA'
    score=9999,  win=438
    ...
  region='NA'
    score=10000, win=500
    ...
```

### Composite 인덱스 사용 가능/불가능 조건

| 쿼리 WHERE/ORDER BY 조건 | 인덱스 사용 여부 | 이유 |
|---|---|---|
| `region = 'NA' AND score > 5000 AND win_count > 10` | 사용 가능 | leading column 포함 |
| `ORDER BY region, score DESC, win_count DESC` | 사용 가능 | 인덱스 정렬 순서와 동일 |
| `ORDER BY region, win_count DESC, score DESC` | **사용 불가** | 정렬 순서가 인덱스 정의와 다름 |
| `score > 1000 AND win_count > 30` (region 없음) | **사용 불가** | leading column 누락 |
| `region = 'EMEA' AND win_count > 30` (score 없음) | 사용 가능 (상황 의존) | leading column 있고 부분 스캔 가능 |

PostgreSQL 18부터 **Skip Scan** 기능이 도입되어, leading column이 누락되어도 Composite B-tree 인덱스를 활용할 수 있는 시나리오가 확대되었다.

---

## 6. Covering 인덱스 (INCLUDE)

```sql
CREATE INDEX idx_composite_covering
  ON game.player_stats (region, score DESC, win_count DESC)
  INCLUDE (username);
```

INCLUDE에 명시된 컬럼은 인덱스 정렬/검색에 사용되지 않고, 단순히 인덱스 Leaf 노드에 값이 저장된다. 이로써 해당 컬럼이 SELECT에 포함되어도 테이블을 조회할 필요가 없어진다.

```
+-------------------------------+
|  Composite Index              |
|  Sorted by: region, score,    |
|             win_count         |
|                               |
|  Leaf Node:                   |
|  [region|score|win_count]     |
|  + stored: [username]  <------+-- INCLUDE column
|  + table ref (ctid)           |
+-------------------------------+
```

적용 전후 비교:

| 항목 | Covering 인덱스 적용 전 | 적용 후 |
|---|---|---|
| Access Method | Bitmap Index Scan → Bitmap Heap Scan | **Index Only Scan** |
| Heap Fetches | 68 pages | **0** |
| Execution Time | ~1.8ms | **~0.6ms (3x 향상)** |

Tradeoff: INCLUDE 컬럼이 변경될 때마다 인덱스도 갱신해야 하므로, 변경 빈도가 낮은 컬럼에 적용하는 것이 좋다.

---

## 7. Partial 인덱스 (조건부 인덱스)

```sql
CREATE INDEX idx_occasional_players
  ON game.player_stats (play_time)
  WHERE play_time <= '50 hours';
```

WHERE 절의 조건을 만족하는 행만 인덱스에 포함된다. 전체 테이블 10,000건 중 조건을 만족하는 74건만 인덱싱하므로 인덱스 크기가 매우 작고, 테이블 갱신 시에도 해당 조건에 해당하는 행만 인덱스를 갱신한다.

성능 차이: Seq Scan ~2ms → Index Scan **~0.1ms (20x 향상)**

주의사항: 쿼리의 WHERE 조건이 인덱스 정의의 WHERE 조건보다 조금이라도 넓으면(예: `play_time <= '50 hours 1 second'`) PostgreSQL은 해당 Partial 인덱스를 사용하지 않고 Seq Scan으로 폴백한다.

---

## 8. Functional / Expression 인덱스

```sql
CREATE INDEX idx_perf_margin
  ON game.player_stats ((win_count - loss_count));
```

컬럼 값 자체가 아니라, 함수나 수식의 결과에 인덱스를 생성한다. 쿼리의 WHERE 또는 ORDER BY 절에 동일한 함수/수식이 포함될 때 플래너가 인덱스를 활용한다.

적용 예시: `lower(username)` 같은 내장 함수나 사용자 정의 함수도 가능하다.

```sql
-- 대소문자 무관 검색 최적화
CREATE INDEX idx_lower_username
  ON game.player_stats (lower(username));

SELECT * FROM game.player_stats
  WHERE lower(username) = 'kelly20';
```

---

## 9. Over-indexing 주의

인덱스는 무료가 아니다. 각 인덱스를 추가할 때마다 발생하는 비용은 다음과 같다.

```
+-----------------------------+
|    Write Operation          |
+-----------------------------+
         |
         v
+--------+---------+
| Update Table Row |
+--------+---------+
         |
         +---> Update Index 1 (idx_score)
         +---> Update Index 2 (idx_champion_title)
         +---> Update Index 3 (idx_composite_covering)
         +---> Update Index 4 (idx_perf_margin)
         +---> ...
         (more indexes = more write overhead)
```

인덱스가 많을수록 INSERT/UPDATE/DELETE 성능이 저하되고, 플래너가 최적 실행 계획을 선택하는 데 걸리는 Planning Time도 증가한다. 현재 테이블의 인덱스 수를 확인하려면 아래 쿼리를 사용한다.

```sql
SELECT indexname, indexdef
  FROM pg_indexes
  WHERE schemaname = 'game'
    AND tablename = 'player_stats';
```

---

## 10. MySQL → PostgreSQL 전환 시 인덱스 관련 핵심 차이 정리

| 항목 | MySQL (InnoDB) | PostgreSQL |
|---|---|---|
| 테이블 구조 | Clustered Index (PK 기반) | Heap Table |
| Secondary Index Leaf | PK 값 저장 | ctid (물리적 행 위치) 저장 |
| Hash Index | Adaptive (자동, 명시 불가) | `USING hash` 명시 생성 |
| Covering Index | MySQL 8.0 이하에서는 제한적 | `INCLUDE` 절로 명시 지원 |
| Partial Index | 미지원 | `WHERE` 절로 지원 |
| Expression Index | MySQL 8.0+ 지원 | 오래전부터 지원 |
| 인덱스 종류 | B-tree, Fulltext, Spatial | B-tree, Hash, GIN, GiST, SP-GiST, BRIN, HNSW, IVFFlat, Bloom, RUM |
| Skip Scan | MySQL 8.0.13+ 지원 | PostgreSQL 18+ 도입 |

---

## Quiz

**Q1.** PostgreSQL에서 인덱스를 생성할 때 `USING` 절을 생략하면 어떤 자료구조가 기본으로 사용되는가?

<details><summary>정답</summary>B-tree가 기본 자료구조로 사용된다.</details>

**Q2.** Hash 인덱스와 B-tree 인덱스의 가장 큰 차이점은 무엇이며, Hash 인덱스를 사용할 수 없는 쿼리 유형은?

<details><summary>정답</summary>Hash 인덱스는 O(1) 복잡도로 등호(=) 및 IN 연산만 지원한다. 범위 검색(>, <, BETWEEN 등)에는 사용할 수 없으며, 이 경우 B-tree를 사용해야 한다.</details>

**Q3.** Composite 인덱스 `(region, score DESC, win_count DESC)`가 정의되어 있을 때, 다음 쿼리가 이 인덱스를 사용할 수 없는 이유를 설명하라.
```sql
SELECT * FROM player_stats
  WHERE score > 1000 AND win_count > 30;
```

<details><summary>정답</summary>Composite 인덱스는 leading column(region)을 기준으로 먼저 정렬되어 있다. 쿼리에 region 조건이 포함되지 않으면 인덱스의 정렬 순서를 활용할 수 없으므로, PostgreSQL 플래너는 해당 인덱스를 사용하지 않고 Seq Scan을 선택한다. (단, PostgreSQL 18+의 Skip Scan 기능이 이 제한을 일부 완화한다.)</details>

**Q4.** `EXPLAIN ANALYZE`와 `EXPLAIN`의 차이를 설명하고, `EXPLAIN (analyze) UPDATE ...` 실행 시 주의해야 할 점은?

<details><summary>정답</summary>EXPLAIN은 실행 계획만 보여주고 쿼리를 실행하지 않는다. EXPLAIN ANALYZE(또는 EXPLAIN (analyze))는 쿼리를 실제로 실행한 뒤 실행 시간 등 실제 메트릭을 포함한 계획을 반환한다. 따라서 UPDATE/DELETE에 사용하면 실제로 데이터가 변경되므로, 트랜잭션 내에서 실행 후 ROLLBACK하는 등의 주의가 필요하다.</details>

**Q5.** Covering 인덱스에서 `INCLUDE (username)` 절의 역할은 무엇이며, 이를 통해 어떤 Access Method가 가능해지는가?

<details><summary>정답</summary>INCLUDE 절에 명시된 컬럼(username)은 인덱스의 정렬이나 검색 조건에는 사용되지 않지만, Leaf 노드에 값이 저장된다. 이를 통해 해당 컬럼이 SELECT에 포함된 쿼리에서도 테이블을 조회하지 않고 Index Only Scan으로 처리할 수 있어 Heap Fetches가 0이 된다.</details>

**Q6.** Partial 인덱스 `WHERE play_time <= '50 hours'`가 정의되어 있을 때, `WHERE play_time <= '50 hours 1 second'` 조건의 쿼리는 이 인덱스를 사용할 수 있는가? 그 이유는?

<details><summary>정답</summary>사용할 수 없다. 쿼리의 검색 범위(`<= 50시간 1초`)가 인덱스가 보장하는 범위(`<= 50시간`)보다 넓기 때문에, PostgreSQL은 인덱스에 포함되지 않은 데이터가 있을 수 있다고 판단하여 Partial 인덱스를 무시하고 Seq Scan으로 폴백한다.</details>