# PostgreSQL 시작하기 — "Just Use Postgres!"의 첫걸음

- 키워드: #PostgreSQL, #Docker, #psql
- 출처: [Just Use Postgres! - Chapter 1: Meeting Postgres (O'Reilly)](https://learning.oreilly.com/library/view/just-use-postgres/9781633435698/Text/chapter-1.html)
- 3줄 요약
  1. PostgreSQL은 오픈소스·엔터프라이즈 안정성·확장성을 기반으로 Stack Overflow 3년 연속(2023-2025) 가장 인기 있는 DB로 선정되었으며, OLTP를 넘어 범용 데이터베이스로 진화하고 있다.
  2. Docker 컨테이너로 1분 이내에 Postgres를 기동할 수 있고, 내장 CLI 도구인 psql로 별도 설치 없이 바로 접속·조작이 가능하다.
  3. `generate_series`, `random` 등 내장 함수만으로 복잡도가 다양한 목업 데이터를 생성할 수 있으며, SQL만으로 집계·정렬·필터 등 핵심 비즈니스 로직을 간결하게 표현할 수 있다.

---

## 1. PostgreSQL이 인기 있는 이유

DB-Engines 랭킹에서 Postgres는 수년간 4위를 유지하면서도 **유일하게 상승 추세**를 보이는 데이터베이스이다. Oracle, MySQL, SQL Server는 정체 또는 하락 중이다.

인기의 3대 요인:

```
+--------------------------------------------------+
|         Why Postgres Is Popular                  |
+--------------------------------------------------+
|                                                  |
|  1. Open Source (MIT License, 1994~)             |
|     - Community-governed                         |
|     - No single vendor lock-in                   |
|                                                  |
|  2. Enterprise-Ready                             |
|     - 35+ years of development                   |
|     - Annual major release, incremental approach |
|     - Fewer regressions between releases         |
|                                                  |
|  3. Extensible by Design                         |
|     - Stonebraker's original design goal         |
|     - Hundreds of extensions                     |
|     - JSON, Time-series, Vector, GIS, FTS ...    |
|                                                  |
+--------------------------------------------------+
```

MySQL 사용자 관점에서 중요한 차이: MySQL은 Oracle이 소유하고 있어 라이선스 정책 변경 리스크가 있지만, Postgres는 PostgreSQL Global Development Group이 관리하는 순수 커뮤니티 프로젝트이다. 확장(Extension) 아키텍처 덕분에 코어를 수정하지 않고도 새로운 데이터 타입, 연산자, 인덱스 접근 방법을 추가할 수 있다.

---

## 2. "Just Use Postgres!" 의 의미

"모든 것에 Postgres를 쓰라"는 뜻이 **아니다**. 핵심 메시지는 다음과 같다:

> 이미 Postgres를 사용 중이라면, 새로운 워크로드(Geospatial, Time-series, GenAI 등)가 생겼을 때 별도 DB를 도입하기 전에 Postgres로 해결 가능한지 먼저 확인하라.

```
+---------------------------+
|   New Use Case Appears    |
+---------------------------+
            |
            v
+---------------------------+
| Can Postgres handle it?   |
+---------------------------+
     |               |
    YES              NO
     |               |
     v               v
+-----------+  +-----------------+
| Just Use  |  | Bring another   |
| Postgres! |  | specialized DB  |
+-----------+  +-----------------+
```

이 접근법의 이점: 운영 복잡도 감소, 학습 비용 절감, 데이터 일관성 유지가 용이해진다.

---

## 3. Docker로 Postgres 시작하기

### 아키텍처 개요

```
+------- Host OS (macOS/Linux/Windows) ---------+
|                                               |
|  +-----------+     +----------------------+   |
|  | Terminal  |---->| Docker Container     |   |
|  +-----------+     |  +----------------+  |   |
|                    |  |   PostgreSQL   |  |   |
|                    |  |   (port 5432)  |  |   |
|                    |  +----------------+  |   |
|                    |  +----------------+  |   |
|                    |  |     psql       |  |   |
|                    |  +----------------+  |   |
|                    +----------|------------+  |
|                               |               |
|                    +----------v-----------+   |
|                    | Docker Volume        |   |
|                    | (postgres-volume)    |   |
|                    | /var/lib/postgresql  |   |
|                    |          /data       |   |
|                    +----------------------+   |
+-----------------------------------------------+
```

### 핵심 명령어

```sql
-- 볼륨 생성 (데이터 영속성 보장)
docker volume create postgres-volume

-- 컨테이너 실행
docker run --name postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=password \
  -p 5432:5432 \
  -v postgres-volume:/var/lib/postgresql/data \
  -d postgres:17.2
```

주요 파라미터 정리:

| 파라미터 | 역할 |
|---------|------|
| `-e POSTGRES_USER` | 기본 사용자명 설정 |
| `-e POSTGRES_PASSWORD` | 기본 비밀번호 설정 |
| `-p 5432:5432` | 컨테이너 포트를 호스트에 매핑 |
| `-v postgres-volume:...` | Docker 볼륨 마운트 — 컨테이너 삭제/재생성 시에도 데이터 유지 |
| `-d` | 백그라운드 실행 |

상태 확인:
```bash
docker container ls -f name=postgres   # STATUS 확인
docker logs postgres                    # "database system is ready to accept connections" 확인
```

---

## 4. psql 접속 및 주요 메타 명령어

```bash
docker exec -it postgres psql -U postgres
```

컨테이너 내부에서 Unix 소켓으로 접속하므로 비밀번호 입력이 불필요하다(trust 인증).

MySQL의 `mysql` CLI와 비교했을 때 psql은 `\` 접두사 메타 명령어 체계를 갖는다:

| psql 명령어 | MySQL 대응 | 설명 |
|------------|-----------|------|
| `\q` | `quit` | 접속 종료 |
| `\conninfo` | `status` | 현재 연결 정보 |
| `\d` | `SHOW TABLES` | 테이블/뷰/시퀀스 목록 |
| `\?` | `\h` | 메타 명령어 전체 목록 |

Postgres에서 "데이터베이스"라는 용어는 테이블, 뷰, 인덱스 등 데이터베이스 오브젝트의 명명된 컬렉션을 의미한다. 초기화 시 `postgres`라는 이름의 기본 데이터베이스가 생성되며, 추가 데이터베이스는 `CREATE DATABASE`로 생성한다.

---

## 5. generate_series를 이용한 목업 데이터 생성

### generate_series 함수 시그니처

```
generate_series(start integer, stop integer [, step integer])
  -> setof integer
```

MySQL에는 이에 대응하는 내장 함수가 없다(MySQL 8.0+에서는 CTE 재귀로 유사하게 구현). Postgres는 이 함수를 통해 정수 또는 타임스탬프 시리즈를 직접 생성할 수 있다.

### 테이블 생성

```sql
CREATE TABLE trades(
    id             bigint,
    buyer_id       integer,
    symbol         text,
    order_quantity integer,
    bid_price      numeric(5,2),
    order_time     timestamp
);
```

MySQL과의 타입 비교: `text`는 MySQL의 `VARCHAR`/`TEXT`에 대응하며, Postgres의 `text`는 길이 제한이 없고 성능 차이도 없다. `numeric(5,2)`는 MySQL의 `DECIMAL(5,2)`과 동일하다.

### 목업 데이터 1,000건 삽입

```sql
INSERT INTO trades (id, buyer_id, symbol, order_quantity, bid_price, order_time)
SELECT
    id,
    random(1,10)                                    AS buyer_id,
    (array['AAPL','F','DASH'])[random(1,3)]         AS symbol,
    random(1,20)                                    AS order_quantity,
    round(random(10.00,20.00), 2)                   AS bid_price,
    now()                                           AS order_time
FROM generate_series(1,1000) AS id;
```

핵심 기법 분석:

```
+------------------------------------------------------------+
|  Data Generation Pipeline (Single SQL Statement)           |
+------------------------------------------------------------+
|                                                            |
|  generate_series(1,1000) ---> Produces 1000 row IDs       |
|       |                                                    |
|       +-- random(1,10)       ---> buyer_id (integer)       |
|       +-- array[...][random] ---> symbol   (text)          |
|       +-- random(1,20)       ---> order_quantity (integer)  |
|       +-- round(random(),2)  ---> bid_price (numeric)      |
|       +-- now()              ---> order_time (timestamp)    |
|       |                                                    |
|       v                                                    |
|  INSERT INTO trades  (1000 rows inserted)                  |
+------------------------------------------------------------+
```

`random(min, max)` 함수는 **Postgres 17에서 추가**되었다. 이전 버전에서는 `floor(random()*(max-min+1)+min)`으로 대체해야 한다.

배열 인덱싱 `(array['A','B','C'])[index]`는 Postgres의 네이티브 배열 기능으로, 텍스트 값을 무작위로 선택하는 간결한 패턴이다. MySQL에는 동등한 배열 타입이 없으므로 `ELT()` 함수로 유사하게 구현한다.

---

## 6. 기본 SQL 쿼리 패턴

### 조건부 카운트

```sql
SELECT count(*) FROM trades WHERE symbol = 'AAPL';
```

`count(*)`는 Postgres에서 특별히 최적화되어 있다. 모든 컬럼 데이터를 실제로 읽지 않고 행 수만 세므로 `count(column_name)`보다 효율적이다. 단, `SELECT *`는 일반적으로 피하는 것이 권장된다 — 필요한 컬럼만 명시하는 습관이 메모리·CPU·네트워크 자원을 절약한다.

### GROUP BY + ORDER BY: 거래량 기준 정렬

```sql
SELECT symbol, count(*) AS total_volume
FROM trades
GROUP BY symbol
ORDER BY total_volume DESC;
```

### 집계 함수 + LIMIT: 상위 N건 추출

```sql
SELECT buyer_id,
       sum(bid_price * order_quantity) AS total_value
FROM trades
GROUP BY buyer_id
ORDER BY total_value DESC
LIMIT 3;
```

이 패턴은 MySQL에서도 거의 동일하게 사용하므로 전환 시 부담이 적다. 다만 Postgres는 `LIMIT` 외에도 SQL 표준인 `FETCH FIRST n ROWS ONLY` 구문을 지원한다.

---

## 7. 이 챕터에서 다루는 Postgres 전체 활용 범위 (이후 챕터 로드맵)

```
+-----------------------------------------------------------+
|              "Just Use Postgres!" Roadmap                 |
+-----------------------------------------------------------+
|                                                           |
|  Part 1: Relational DB                                    |
|    Ch1  Meeting Postgres          <-- YOU ARE HERE        |
|    Ch2  Standard RDBMS                                    |
|    Ch3  Modern SQL                                        |
|    Ch4  Indexes                                           |
|                                                           |
|  Part 2: Beyond Relational                                |
|    Ch5  JSON support                                      |
|    Ch6  Full-Text Search                                  |
|                                                           |
|  Part 3: Extensions & Ecosystem                           |
|    Ch7  Extensions overview                               |
|    Ch8  Generative AI  (pgvector)                         |
|    Ch9  Time-Series    (TimescaleDB)                      |
|    Ch10 Geospatial     (PostGIS)                          |
|    Ch11 Message Queue  (pgmq/LISTEN-NOTIFY)               |
|                                                           |
|  Appendix A: 5 Optimization Tips                          |
|  Appendix B: When NOT to use Postgres                     |
+-----------------------------------------------------------+
```

AI Engineer 커리어 관점에서 특히 주목할 챕터: **Ch8 (pgvector를 활용한 벡터 유사도 검색)** — RAG 파이프라인, 임베딩 저장소 등에 직접 활용 가능하다.

---

## Quiz

**Q1.** Postgres의 인기를 뒷받침하는 3대 요인을 서술하시오.

<details>
<summary>정답 보기</summary>
(1) 오픈소스이며 커뮤니티가 관리 (MIT 라이선스, PostgreSQL Global Development Group 주도)
(2) 35년 이상의 개발 역사를 가진 엔터프라이즈급 안정성 (연간 메이저 릴리스, 점진적 개선 방식)
(3) 설계 단계부터 확장성(Extensibility)을 핵심 목표로 반영 — 데이터 타입, 연산자, 인덱스 접근 방법을 사용자가 확장 가능
</details>

**Q2.** Docker로 Postgres를 실행할 때 `-v postgres-volume:/var/lib/postgresql/data` 옵션의 역할은 무엇이며, 이 옵션이 없을 경우 어떤 문제가 발생하는가?

<details>
<summary>정답 보기</summary>
Docker 볼륨을 컨테이너의 Postgres 데이터 디렉토리에 마운트하여, 컨테이너를 중지·삭제·재생성하더라도 데이터가 호스트에 영속적으로 보존된다. 이 옵션이 없으면 컨테이너 삭제 시 모든 데이터베이스 데이터가 함께 소실된다.
</details>

**Q3.** psql에서 컨테이너 내부 접속 시 비밀번호를 묻지 않는 이유를 설명하시오.

<details>
<summary>정답 보기</summary>
`docker exec -it`로 컨테이너 내부에 진입한 뒤 psql을 실행하면, 로컬 Unix 소켓을 통해 연결된다. Postgres Docker 이미지는 이러한 로컬 소켓 연결을 trust 인증으로 설정하고 있어 비밀번호 입력이 필요 없다.
</details>

**Q4.** 다음 SQL에서 `(array['AAPL','F','DASH'])[random(1,3)]`이 하는 역할을 설명하고, MySQL에서 동일한 동작을 구현하는 방법을 제시하시오.

```sql
SELECT id, (array['AAPL','F','DASH'])[random(1,3)] AS symbol
FROM generate_series(1,5) AS id;
```

<details>
<summary>정답 보기</summary>
3개의 텍스트 요소를 가진 Postgres 배열을 선언하고, random(1,3)으로 생성한 1~3 범위의 정수를 인덱스로 사용하여 무작위로 하나의 주식 심볼을 선택한다.

MySQL 대응:
```sql
SELECT id, ELT(FLOOR(1 + RAND() * 3), 'AAPL', 'F', 'DASH') AS symbol
FROM (SELECT @row := @row + 1 AS id
      FROM information_schema.tables,
           (SELECT @row := 0) r LIMIT 5) t;
```
또는 MySQL 8.0+에서 CTE 재귀를 사용할 수 있다.
</details>

**Q5.** `count(*)`와 `count(column_name)`의 차이를 Postgres 최적화 관점에서 설명하시오.

<details>
<summary>정답 보기</summary>
`count(*)`는 Postgres에서 특별히 최적화되어 있어, 실제 컬럼 데이터를 읽지 않고 행의 존재 여부만 확인하여 카운트한다. 반면 `count(column_name)`은 해당 컬럼 값이 NULL이 아닌 행만 세므로 컬럼 데이터를 실제로 읽어야 하며, NULL 체크 비용이 추가된다.
</details>

**Q6.** "Just Use Postgres"의 의미를 올바르게 설명한 것은?

- A) 모든 워크로드에 반드시 Postgres를 사용해야 한다
- B) Postgres가 세계에서 가장 빠른 데이터베이스이다
- C) 이미 Postgres를 사용 중이라면, 새 워크로드 발생 시 다른 DB를 추가하기 전에 Postgres로 해결 가능한지 먼저 검토하라는 의미이다
- D) MySQL보다 항상 우수하다는 의미이다

<details>
<summary>정답 보기</summary>
C. Postgres가 범용 DB로서 다양한 워크로드를 처리할 수 있으므로, 불필요하게 기술 스택을 복잡하게 만들기 전에 Postgres의 가능성을 먼저 확인하라는 실용적 조언이다. 해결이 어려우면 다른 전문 DB를 도입하면 된다.
</details>