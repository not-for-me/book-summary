# Modern SQL in PostgreSQL — CTE, Recursive Query, Window Function

- 키워드: CTE (Common Table Expression), Recursive Query, Window Function, WITH clause, PARTITION BY, OVER, RANK(), UNION ALL
- 출처: [Just Use Postgres! — Chapter 3: Modern SQL](https://learning.oreilly.com/library/view/just-use-postgres/9781633435698/Text/chapter-3.html)
- 3줄 요약
  1. CTE(WITH 절)를 사용하면 복잡한 쿼리를 읽기 쉽고 관리하기 편한 단위로 분리할 수 있으며, PostgreSQL은 내부적으로 CTE를 최적화(fold)하여 성능 저하 없이 활용할 수 있다.
  2. Recursive Query(WITH RECURSIVE)는 계층형·트리형 데이터를 SQL만으로 탐색·집계할 수 있게 해주며, working table과 intermediate table을 반복하며 결과를 누적한다.
  3. Window Function(OVER 절)은 GROUP BY와 달리 개별 행을 유지하면서 집계·순위·누적 합계를 계산할 수 있어, self-join 없이도 복잡한 분석이 가능하다.

---

## 1. Modern SQL이란?

SQL-92 표준 이후 추가된 기능 범주를 통칭한다. 관계형 모델에 국한되지 않고 배열, JSON, 재귀 쿼리, 윈도우 함수 등을 포함한다. Markus Winand에 따르면 관계형 SQL은 현재 SQL 표준의 약 20%에 불과하다.

MySQL 사용자 관점에서 주목할 차이점: MySQL 8.0부터 CTE와 Window Function을 지원하지만 PostgreSQL은 8.4(2009)부터 Window Function, Recursive CTE를 지원해 왔으며 `MATERIALIZED` / `NOT MATERIALIZED` 힌트, data-modifying CTE(`INSERT/UPDATE/DELETE … RETURNING`) 등 MySQL에는 없거나 제한적인 기능이 있다.

---

## 2. CTE (Common Table Expression)

### 2.1 기본 구조

```sql
WITH cte_name AS (
    auxiliary_statement   -- 보조 쿼리 (SELECT / INSERT / UPDATE / DELETE)
)
primary_statement;        -- CTE 결과를 참조하는 최종 쿼리
```

CTE는 해당 쿼리 내에서만 유효한 **임시 테이블(또는 뷰)** 과 같다.

### 2.2 SELECT CTE 예시 — 인기곡 집계

```sql
WITH plays_cte AS (
    SELECT s.title, s.duration
    FROM streaming.plays p
    JOIN streaming.songs s ON p.song_id = s.id
    WHERE p.play_start_time::DATE BETWEEN '2024-09-15' AND '2024-09-16'
      AND p.play_duration = s.duration
)
SELECT title, COUNT(*) AS play_count
FROM plays_cte
GROUP BY title
ORDER BY play_count DESC;
```

### 2.3 다중 CTE

하나의 쿼리에 여러 CTE를 정의할 수 있으며, 후속 CTE는 앞서 정의된 CTE를 참조할 수 있다.

```
+----------------+       +--------------------+       +------------------+
|  plays_cte     | ----> | user_play_counts   | ----> | primary SELECT   |
| (filter plays) |       | (count per user)   |       | (final filter)   |
+----------------+       +--------------------+       +------------------+
```

### 2.4 CTE 최적화 — PostgreSQL의 Fold 동작

PostgreSQL은 CTE를 반드시 별도로 실행하지 않는다. 옵티마이저가 CTE를 primary statement에 **fold**(인라인 전개)하여 단일 실행 계획으로 합칠 수 있다. `EXPLAIN ANALYZE`로 확인 가능하다.

| 키워드 | 동작 |
|---|---|
| (기본) | 1회만 참조 시 fold, 다중 참조 시 materialize |
| `AS MATERIALIZED` | 강제로 한 번 실행 후 결과 캐시 |
| `AS NOT MATERIALIZED` | 참조할 때마다 재평가 |

MySQL과 비교: MySQL 8.0 CTE는 기본적으로 항상 materialize되며, `NOT MATERIALIZED` 힌트를 지원하지 않는다. PostgreSQL이 더 유연하다.

### 2.5 Data-Modifying CTE (PostgreSQL 고유 강점)

CTE 안에서 `INSERT`, `UPDATE`, `DELETE`를 수행하고 `RETURNING` 절로 변경된 행을 primary statement에 전달할 수 있다.

```sql
WITH updated_play AS (
    UPDATE streaming.plays
    SET play_duration = 200
    WHERE id = 30
    RETURNING song_id, play_duration
)
SELECT s.title, s.duration,
    CASE WHEN up.play_duration = s.duration
         THEN 'Moved Up the Rank'
         ELSE 'Rank Not Changed'
    END AS rank_change_status
FROM updated_play up
JOIN streaming.songs s ON s.id = up.song_id;
```

핵심 규칙:

- Data-modifying CTE는 **primary statement가 참조하지 않아도 실행**된다 (SELECT CTE와 반대).
- 같은 쿼리 내 여러 CTE는 **동일 스냅샷에서 동시 실행**되므로, 한 CTE의 변경이 다른 CTE에 보이지 않는다.
- 후속 CTE가 앞선 CTE의 변경을 보려면, 테이블이 아닌 **해당 CTE를 직접 참조**해야 한다.

```
Concurrent execution (same snapshot)
+---------------------+     +-------------------------+
| updated_play (UPD)  |     | current_play (SELECT)   |
| play_duration = 150 |     | FROM streaming.plays    |
+---------------------+     | => sees OLD value       |
                             +-------------------------+

Sequential execution (CTE references CTE)
+---------------------+     +-------------------------+
| updated_play (UPD)  | --> | current_play (SELECT)   |
| play_duration = 160 |     | FROM updated_play       |
+---------------------+     | => sees NEW value       |
                             +-------------------------+
```

---

## 3. Recursive Query (WITH RECURSIVE)

### 3.1 구조

```sql
WITH RECURSIVE cte_name (col1, col2, ...) AS (
    -- Non-recursive term (1회 실행, 초기 데이터)
    SELECT ... FROM table WHERE condition
    UNION [ALL]
    -- Recursive term (working table이 빌 때까지 반복)
    SELECT ... FROM table JOIN cte_name ON recursive_condition
)
SELECT ... FROM cte_name;
```

| 구성 요소 | 역할 |
|---|---|
| Non-recursive term | 시작점(루트) 데이터를 working table에 적재 |
| UNION ALL / UNION | 결과 병합. UNION은 중복 제거, UNION ALL은 전체 포함 |
| Recursive term | working table을 입력으로 사용, 결과를 intermediate table에 저장 |
| 종료 조건 | recursive term이 빈 결과를 반환하면 종료 |

### 3.2 실행 흐름 (Pseudocode 기반)

```
+---------------------------+
| Non-recursive term        |
| => working_table          |
| => final_result += result |
+---------------------------+
            |
            v
+---------------------------+
| WHILE working_table       |
|       is NOT empty:       |
|                           |
|  intermediate = execute(  |
|    recursive_term,        |
|    using working_table    |
|  )                        |
|                           |
|  if UNION:                |
|    deduplicate            |
|                           |
|  final_result +=          |
|    intermediate           |
|                           |
|  working_table =          |
|    intermediate           |
+---------------------------+
            |
            v
+---------------------------+
| Return final_result       |
+---------------------------+
```

### 3.3 예시 — 연속 재생 시퀀스 탐색

`streaming.plays` 테이블에서 `played_after` 컬럼이 이전 재생 세션의 id를 가리키는 linked-list 구조이다.

```sql
WITH RECURSIVE play_sequence AS (
    -- root: id=5인 세션에서 시작
    SELECT id, user_id, song_id, play_start_time,
           play_duration, played_after, 1 AS level
    FROM streaming.plays
    WHERE id = 5
    UNION ALL
    -- 다음 곡 탐색
    SELECT p.id, p.user_id, p.song_id, p.play_start_time,
           p.play_duration, p.played_after, level + 1
    FROM streaming.plays p
    JOIN play_sequence ps ON p.played_after = ps.id
)
SELECT * FROM play_sequence ORDER BY play_start_time;
```

결과 구조:

```
level 1:  id=5  song=5  played_after=NULL
level 2:  id=6  song=6  played_after=5
level 3:  id=7  song=7  played_after=6
level 4:  id=8  song=8  played_after=7
```

### 3.4 재귀 인자 활용 — 누적 재생 시간 & 경로 추적

CTE 정의에 `(parent_id, sequence, total_duration)` 같은 인자를 선언하면 재귀 단계마다 값을 누적할 수 있다.

```sql
WITH RECURSIVE play_sequence(parent_id, sequence, total_duration) AS (
    SELECT id, ARRAY[id], play_duration
    FROM streaming.plays WHERE id = 5
    UNION ALL
    SELECT p.id,
           ps.sequence || p.id,          -- 배열에 현재 id 추가
           total_duration + p.play_duration  -- 누적 합산
    FROM streaming.plays p
    JOIN play_sequence ps ON p.played_after = ps.parent_id
)
SELECT * FROM play_sequence;
```

PostgreSQL의 `ARRAY` 타입을 활용해 경로를 배열로 관리하는 것은 MySQL에서 쉽게 구현하기 어려운 패턴이다. MySQL의 Recursive CTE는 배열 타입이 없어 `CONCAT` 등 문자열 처리로 대체해야 한다.

---

## 4. Window Function

### 4.1 기본 구조

```sql
SELECT function_name(args)
       OVER (PARTITION BY colA ORDER BY colB)
FROM table;
```

| 절 | 역할 |
|---|---|
| `OVER (...)` | 윈도우 함수임을 선언 |
| `PARTITION BY` | 데이터를 윈도우(그룹)로 분할. 생략 시 전체가 하나의 윈도우 |
| `ORDER BY` | 윈도우 내 행 정렬 → frame 생성 → 누적 계산 가능 |

### 4.2 GROUP BY vs Window Function 핵심 차이

```
GROUP BY                          Window Function
+-----------+                     +-----------+
| song_id=1 | => 1 row            | song_id=1 | => N rows (per user)
| SUM=800   |                     | user=1 SUM=800 |
+-----------+                     | user=2 SUM=800 |
                                  | user=3 SUM=800 |
                                  +-----------+
```

GROUP BY는 그룹당 하나의 행만 반환하지만, Window Function은 **모든 개별 행을 유지**하면서 집계 결과를 함께 출력한다.

### 4.3 self-join 제거 효과

Window Function 없이 "곡별 전체 재생시간 + 사용자별 행"을 구하려면 self-join이 필요하다.

```sql
-- self-join 방식 (테이블 2회 스캔)
SELECT DISTINCT p.song_id, p.user_id, t.total_duration
FROM streaming.plays p
JOIN (
    SELECT song_id, SUM(play_duration) AS total_duration
    FROM streaming.plays GROUP BY song_id
) t ON p.song_id = t.song_id;

-- Window Function 방식 (테이블 1회 스캔)
SELECT DISTINCT song_id, user_id,
       SUM(play_duration) OVER (PARTITION BY song_id) AS total_duration
FROM streaming.plays;
```

Window Function 방식이 테이블을 한 번만 스캔하므로 더 효율적이다.

### 4.4 Running Total (누적 합계)

`OVER` 절에 `ORDER BY`를 추가하면 윈도우 내에서 frame 단위 누적 계산이 이루어진다.

```sql
SELECT song_id, user_id, play_duration,
       SUM(play_duration) OVER (PARTITION BY song_id ORDER BY user_id)
         AS running_total
FROM streaming.plays
WHERE song_id = 2;
```

```
+-------------------------------------------+
| Window: song_id = 2                       |
|                                           |
| Frame 1: user=1  144      -> total= 144  |
| Frame 2: user=2  206      -> total= 350  |
| Frame 3: user=3  186,118  -> total= 654  |
+-------------------------------------------+
```

### 4.5 RANK() — 순위 함수

```sql
SELECT song_id,
       SUM(play_duration) AS total_play_duration,
       RANK() OVER (ORDER BY SUM(play_duration) DESC) AS song_rank
FROM streaming.plays
GROUP BY song_id
ORDER BY song_rank;
```

주요 Window-only 함수 정리:

| 함수 | 설명 |
|---|---|
| `ROW_NUMBER()` | 파티션 내 고유 순번 (동순위 없음) |
| `RANK()` | 동순위 허용, 다음 순위 건너뜀 (1,2,2,4) |
| `DENSE_RANK()` | 동순위 허용, 다음 순위 연속 (1,2,2,3) |
| `LAG(col, n)` | 현재 행 기준 n행 이전 값 |
| `LEAD(col, n)` | 현재 행 기준 n행 이후 값 |
| `FIRST_VALUE(col)` | 윈도우/프레임 첫 번째 값 |
| `LAST_VALUE(col)` | 윈도우/프레임 마지막 값 |
| `NTILE(n)` | 파티션을 n개 버킷으로 균등 분할 |

---

## 5. MySQL → PostgreSQL 전환 시 주의 포인트 요약

| 항목 | MySQL 8.0+ | PostgreSQL |
|---|---|---|
| CTE Optimization | 기본 materialize | 기본 fold(inline), 명시적 MATERIALIZED/NOT MATERIALIZED 지원 |
| Data-Modifying CTE | 미지원 | INSERT/UPDATE/DELETE + RETURNING 가능 |
| Recursive CTE | 지원 (UNION ALL만 안정) | UNION / UNION ALL 모두 지원, ARRAY 경로 추적 가능 |
| ARRAY 타입 | 미지원 | 네이티브 지원 (`ARRAY[1,2,3]`, `||` 연산) |
| Window Function | 지원 | 동일 + 더 많은 frame 옵션 |

---

## Quiz

**Q1.** CTE를 정의하는 SQL 키워드는 무엇이며, CTE의 결과는 쿼리 외부에서도 재사용할 수 있는가?

<details>
<summary>정답</summary>
WITH 절을 사용한다. CTE 결과는 해당 쿼리 내에서만 유효하며 쿼리 외부에서는 재사용할 수 없다 (임시 테이블/뷰와 유사).
</details>

**Q2.** PostgreSQL에서 Data-Modifying CTE(UPDATE … RETURNING)를 사용할 때, 같은 쿼리 내 다른 CTE가 변경 사항을 보려면 어떻게 해야 하는가? 그리고 그 이유는?

<details>
<summary>정답</summary>
후속 CTE가 원본 테이블이 아닌 Data-Modifying CTE 자체를 직접 참조해야 한다. 같은 쿼리의 CTE들은 동일한 스냅샷에서 동시 실행되므로, 테이블을 직접 조회하면 변경 전의 데이터가 보인다.
</details>

**Q3.** Recursive CTE에서 UNION과 UNION ALL의 차이점은 무엇이며, 재귀가 종료되는 조건은?

<details>
<summary>정답</summary>
UNION은 각 재귀 단계에서 중복 행을 제거하고, UNION ALL은 모든 행을 포함한다. 재귀는 recursive term이 빈 결과(0행)를 반환하면 종료된다.
</details>

**Q4.** Window Function에서 `PARTITION BY`를 생략하면 어떤 일이 발생하는가? 또한 `ORDER BY`를 `OVER` 절에 추가하면 결과가 어떻게 달라지는가?

<details>
<summary>정답</summary>
PARTITION BY를 생략하면 전체 결과 집합이 하나의 윈도우로 처리된다. OVER 절에 ORDER BY를 추가하면 윈도우 내에서 frame이 생성되어 running total(누적 합계) 형태의 계산이 이루어진다.
</details>

**Q5.** `RANK()`와 `DENSE_RANK()`의 차이를 총 재생시간이 같은 두 곡이 있다고 가정하고 설명하라.

<details>
<summary>정답</summary>
총 재생시간이 같은 두 곡이 2위에 해당한다면, RANK()는 둘 다 2위를 부여하고 다음 순위를 4위로 건너뛴다 (1, 2, 2, 4). DENSE_RANK()는 둘 다 2위를 부여하되 다음 순위를 3위로 연속 부여한다 (1, 2, 2, 3).
</details>

**Q6.** PostgreSQL이 CTE를 fold(인라인 전개)하지 않고 반드시 materialize하는 경우는 언제인가? 또한 강제로 materialization 동작을 제어하려면 어떤 키워드를 사용하는가?

<details>
<summary>정답</summary>
CTE가 primary statement 또는 다른 CTE에서 2회 이상 참조되면 PostgreSQL은 자동으로 materialize한다(한 번 실행 후 결과를 캐시). 강제로 제어하려면 `AS MATERIALIZED` (항상 캐시) 또는 `AS NOT MATERIALIZED` (매번 재평가)를 CTE 정의에 추가한다.
</details>