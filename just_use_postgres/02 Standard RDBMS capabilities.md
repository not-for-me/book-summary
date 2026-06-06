# PostgreSQL 표준 RDBMS 기능 — 멀티테넌트 eCommerce 플랫폼 설계를 통한 실전 학습

- **키워드**: Database/Schema/Table 계층구조, DML(SELECT/INSERT/UPDATE/DELETE), Constraints, Foreign Key, Transaction(ACID/MVCC), JOIN, Function/Trigger(PL/pgSQL), View/Materialized View, Role & Access Control
- **출처**: [Just Use Postgres! - Chapter 2: Standard RDBMS capabilities](https://learning.oreilly.com/library/view/just-use-postgres/9781633435698/Text/chapter-2.html)
- **3줄 요약**:
  - PostgreSQL은 Database → Schema → Table의 3단계 계층으로 데이터를 구조화하며, 멀티테넌트 아키텍처 구현에 적합한 격리 수준을 제공한다.
  - Constraint, Transaction(MVCC 기반), Function/Trigger, View를 활용하여 데이터 무결성과 비즈니스 로직을 DB 레벨에서 직접 처리할 수 있다.
  - Role 기반 접근 제어를 통해 테넌트별 데이터 격리와 세분화된 권한 관리가 가능하다.

---

## 1. 데이터베이스 구조: Database → Schema → Table

PostgreSQL은 3단계 계층 구조로 데이터를 조직한다. MySQL과의 가장 큰 차이점 중 하나는 **Schema**라는 중간 계층이 존재한다는 것이다.

![[Pasted image 20260228145541.png]]
```
+-------------------------------------------------------+
|                    Postgres Instance                  |
|                                                       |
|  +-----------------------+  +-----------------------+ |
|  |  DB: coffee_chain     |  |  DB: brewery          | |
|  |                       |  |                       | |
|  | +--------+ +--------+ |  | +--------+ +-------+  | |
|  | |Schema  | |Schema  | |  | |Schema  | |Schema |  | |
|  | |products| |sales   | |  | |products| |sales  |  | |
|  | |        | |        | |  | |        | |       |  | |
|  | |catalog | |orders  | |  | |catalog | |orders |  | |
|  | |reviews | |order_  | |  | |        | |       |  | |
|  | |        | |items   | |  | +--------+ +-------+  | |
|  | +--------+ +--------+ |  |                       | |
|  |                       |  | +-------+             | |
|  | +---------+           |  | |Schema |             | |
|  | |Schema   |           |  | |public |             | |
|  | |customers|           |  | +-------+             | |
|  | |accounts |           |  +-----------------------+ |
|  | +---------+           |                            |
|  +-----------------------+                            |
+-------------------------------------------------------+
```

**핵심 규칙**: 같은 Database 내의 서로 다른 Schema에 있는 테이블은 하나의 SQL 쿼리로 JOIN 가능하다. 그러나 서로 다른 Database에 걸친 cross-database 쿼리는 기본적으로 불가능하다(3rd-party 확장 필요).

**MySQL과의 비교**: MySQL에서 `database`와 `schema`는 동의어이다. PostgreSQL에서는 명확히 분리된 계층이다.

### 1.1 Database 생성

```sql
CREATE DATABASE coffee_chain;
CREATE DATABASE brewery;

-- 목록 확인
SELECT datname FROM pg_database;
```

초기화 시 `postgres`라는 기본 database가 생성되며, 사용자는 기본적으로 이 database에 접속한다.


## Chapter 2에서 사용된 주요 커맨드 정리

### psql 메타 커맨드 (backslash commands)

| 커맨드                | 설명                             | 사용 예시                 |
| ------------------ | ------------------------------ | --------------------- |
| `\c database_name` | 지정한 database로 접속 전환            | `\c coffee_chain`     |
| `\dn`              | 현재 database의 schema 목록 조회      | `\dn`                 |
| `\d`               | 현재 schema의 테이블, 시퀀스 등 객체 목록 조회 | `\d`                  |
| `\d table_name`    | 특정 테이블의 컬럼, 타입, 제약조건 상세 조회     | `\d products.catalog` |
| `\dts schema.*`    | 특정 schema 내 테이블 목록 조회          | `\dts products.*`     |
| `\du`              | 전체 Role 목록과 속성(Superuser 등) 조회 | `\du`                 |
| `\l`               | 전체 database 목록과 소유자 조회         | `\l`                  |

### 1.2 Schema 생성

```sql
-- coffee_chain DB에 접속 후
CREATE SCHEMA products;
CREATE SCHEMA customers;
CREATE SCHEMA sales;
```

모든 database는 기본적으로 `public` schema를 갖는다. 별도 schema를 지정하지 않으면 모든 객체는 `public`에 생성된다.

**search_path**: 현재 활성 schema를 결정하는 설정이다.

```sql
SHOW search_path;       -- 기본값: "$user", public
SET search_path TO products;  -- 현재 schema 변경
```

### 1.3 Table 생성과 데이터 타입

```sql
CREATE TABLE products.catalog (
    id            SERIAL PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    description   TEXT NOT NULL,
    category      TEXT CHECK (category IN ('coffee', 'mug', 't-shirt')),
    price         NUMERIC(10, 2),
    stock_quantity INT CHECK (stock_quantity >= 0)
);
```

**자동 ID 생성 타입 비교**:

| 타입 | 비트 | 최대값 | 용도 |
|------|------|--------|------|
| `SERIAL` | 32-bit | 2,147,483,647 (2³¹−1) | 일반적인 테이블 |
| `BIGSERIAL` | 64-bit | 9,223,372,036,854,775,807 (2⁶³−1) | 대량 레코드 테이블 |
| `UUID` (gen_random_uuid()) | 128-bit | - | 분산 환경, Sequence 의존 제거 |

**참고**: PostgreSQL 18부터 `uuidv7()` 함수가 도입되었다. UUIDv7은 Unix timestamp를 포함하여 인덱스 친화적이다(시간순으로 가까운 UUID가 인덱스에서 인접 저장).

---

## 2. 데이터 조작(DML)

MySQL과 동일한 표준 SQL DML을 사용하지만, schema 접두어를 사용하는 점이 다르다.

```sql
-- INSERT
INSERT INTO products.catalog (name, description, category, price, stock_quantity)
VALUES ('Sunrise Blend', 'A smooth blend...', 'coffee', 14.99, 50);

-- SELECT with WHERE
SELECT id, name, price FROM products.catalog WHERE category = 'coffee';

-- UPDATE
UPDATE products.catalog SET price = 16.54 WHERE id = 1;

-- DELETE
DELETE FROM products.catalog WHERE id = 2;
```

---

## 3. 데이터 무결성(Data Integrity)

### 3.1 Constraint 종류

| Constraint | 설명 |
|-----------|------|
| NOT NULL | NULL 값 불허 |
| UNIQUE | 중복 값 불허 |
| PRIMARY KEY | NOT NULL + UNIQUE (행의 고유 식별자) |
| FOREIGN KEY | 다른 테이블의 값 참조 보장 |
| CHECK | 사용자 정의 조건 검증 |
| EXCLUSION | 지정 연산자로 비교 시 두 행이 동시에 만족 불가 보장 |

기존 테이블에 constraint 추가:

```sql
-- CHECK constraint 추가
ALTER TABLE products.catalog
ADD CONSTRAINT catalog_price_check CHECK (price > 0);

-- NOT NULL 추가 + CHECK 동시 추가
ALTER TABLE products.reviews
ALTER COLUMN review SET NOT NULL,
ADD CONSTRAINT review_rank_check CHECK (rank BETWEEN 1 AND 5);
```

**주의**: `ALTER TABLE`로 constraint를 추가할 때, 기존 데이터 중 위반하는 행이 있으면 실패한다. 먼저 위반 데이터를 수정한 후 constraint를 적용해야 한다.

### 3.2 Foreign Key

```sql
-- reviews.product_id → catalog.id 참조
ALTER TABLE products.reviews
ADD CONSTRAINT products_review_product_id_fk
FOREIGN KEY (product_id) REFERENCES products.catalog(id);

-- reviews.customer_id → accounts.id 참조
ALTER TABLE products.reviews
ADD CONSTRAINT products_review_customer_id_fk
FOREIGN KEY (customer_id) REFERENCES customers.accounts(id);
```

Foreign Key의 **양방향 보호**:

```
+------------------+        FK        +------------------+
| products.catalog | <--------------- | products.reviews |
|   id (PK)        |                  |   product_id(FK) |
+------------------+                  +------------------+

  1) INSERT reviews with invalid product_id -> ERROR
  2) DELETE catalog row referenced by reviews -> ERROR
```

참조되는 행을 삭제하려 하면 에러가 발생한다. 이를 해결하는 두 가지 패턴:

- **Soft Delete**: `deleted BOOLEAN DEFAULT false` 컬럼을 추가하고 flag만 변경
- **CASCADE Delete**: FK 정의 시 `ON DELETE CASCADE` 추가 → 부모 삭제 시 자식도 함께 삭제

---

## 4. Transaction

### 4.1 암시적(Implicit) vs 명시적(Explicit) Transaction

PostgreSQL은 **모든 단일 DML 문을 암시적 트랜잭션**으로 실행한다. 여러 문을 하나의 원자적 단위로 묶으려면 명시적 트랜잭션을 사용한다.

```sql
BEGIN;
  INSERT INTO sales.orders (...) VALUES (...);
  INSERT INTO sales.order_items (...) VALUES (...), (...);
  UPDATE products.catalog SET stock_quantity = stock_quantity - 1 WHERE id IN (1, 4);
COMMIT;
-- 어떤 단계에서든 실패하면 전체 ROLLBACK
```

### 4.2 MVCC (Multi-Version Concurrency Control)

PostgreSQL의 동시성 제어 핵심 메커니즘이다. MySQL(InnoDB)도 MVCC를 사용하지만 구현 방식이 다르다.

```
   Session A                         Session B
   --------                          --------
   BEGIN;
   SELECT stock_quantity ...  -> 199
   UPDATE ... SET stock_quantity
        = stock_quantity - 1;
                                     BEGIN;
                                     SELECT stock_quantity ... -> 199
                                       (A의 미커밋 변경은 보이지 않음)
                                     UPDATE ... SET stock_quantity
                                          = stock_quantity - 1;
                                       ** BLOCKED ** (A가 같은 행 잠금)
   COMMIT;
                                       (Unblocked: 최신값 198 기반으로 재계산)
                                     COMMIT;
   -- 최종 결과: 197 (199 - 1 - 1)
```

**기본 격리 수준**: `Read Committed` — 다른 트랜잭션의 미커밋 변경(Dirty Read)을 방지한다.

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|-----------|-----------|-------------------|-------------|
| Read Committed (기본) | 방지 | 가능 | 가능 |
| Repeatable Read | 방지 | 방지 | 방지 |
| Serializable | 방지 | 방지 | 방지 |

**PostgreSQL 특이사항**: ROLLBACK된 변경은 즉시 물리적으로 삭제되지 않는다. MVCC 엔진에 의해 보이지 않게 되지만, 물리적 제거는 **VACUUM**이 수행한다.

---

## 5. JOIN

| JOIN 유형 | 설명 |
|----------|------|
| INNER JOIN | 양쪽 테이블 모두에서 매칭되는 행만 반환 |
| LEFT JOIN | 왼쪽 테이블 전체 + 오른쪽 매칭 (없으면 NULL) |
| RIGHT JOIN | 오른쪽 테이블 전체 + 왼쪽 매칭 |
| FULL OUTER JOIN | 양쪽 전체 + 매칭 안 되는 행은 NULL |

```sql
-- INNER JOIN: 주문 수 기준 상위 3 고객
SELECT c.name, count(*) as total_orders
FROM customers.accounts c
JOIN sales.orders s ON c.id = s.customer_id
GROUP BY c.id
ORDER BY total_orders DESC
LIMIT 3;

-- LEFT JOIN + NULLS LAST: 제품별 판매 현황
SELECT c.name, SUM(oi.quantity) AS total_sold
FROM products.catalog c
LEFT JOIN sales.order_items oi ON c.id = oi.product_id
GROUP BY c.id
ORDER BY total_sold DESC NULLS LAST;
```

`NULLS LAST` / `NULLS FIRST`: PostgreSQL에서 NULL 정렬 위치를 명시적으로 제어하는 구문이다.

---

## 6. Function & Trigger

### 6.1 Function (PL/pgSQL)

PostgreSQL 함수는 **원자적이고 트랜잭셔널**하게 실행된다. 중간에 실패하면 함수 내 모든 변경이 롤백된다.

```sql
CREATE OR REPLACE FUNCTION sales.order_add_item(
    customer_id_param INT,
    product_id_param INT,
    quantity_param INT
) RETURNS TABLE (order_id UUID, prod_id INT, quantity INT, prod_price DECIMAL)
AS $$
DECLARE
    pending_order_id UUID;
BEGIN
    -- 1. 기존 pending 주문 검색
    SELECT id INTO pending_order_id
    FROM sales.orders
    WHERE customer_id = customer_id_param AND status = 'pending'
    LIMIT 1;

    -- 2. 없으면 새 주문 생성
    IF pending_order_id IS NULL THEN
        INSERT INTO sales.orders (customer_id, status)
        VALUES (customer_id_param, 'pending')
        RETURNING id INTO pending_order_id;
    END IF;

    -- 3. MERGE로 상품 추가/수량 갱신
    MERGE INTO sales.order_items AS oi
    USING (SELECT id, price FROM products.catalog
           WHERE id = product_id_param) AS prod
    ON oi.product_id = prod.id AND oi.order_id = pending_order_id
    WHEN MATCHED THEN
        UPDATE SET quantity = quantity_param
    WHEN NOT MATCHED THEN
        INSERT (order_id, product_id, quantity, price)
        VALUES (pending_order_id, prod.id, quantity_param, prod.price);

    -- 4. 현재 주문 아이템 목록 반환
    RETURN QUERY SELECT ...;
END;
$$ LANGUAGE plpgsql;
```

**핵심 구문 정리**:

| 구문 | 설명 |
|-----|------|
| `DECLARE` | 변수 선언 블록 |
| `SELECT ... INTO` | 쿼리 결과를 변수에 할당 |
| `RETURNING ... INTO` | INSERT/UPDATE 결과 값을 변수에 캡처 |
| `MERGE INTO ... WHEN MATCHED / NOT MATCHED` | UPSERT 패턴 (PostgreSQL 15+) |
| `RETURN QUERY` | 결과 집합 반환 |
| `RAISE EXCEPTION` | 에러 발생 |

**호출 방식**:
```sql
-- Named notation
SELECT * FROM sales.order_add_item(
    customer_id_param => 3, product_id_param => 3, quantity_param => 2);
-- Positional notation
SELECT * FROM sales.order_add_item(3, 3, 2);
```

**참고**: PostgreSQL에서 `FUNCTION`과 `PROCEDURE`는 다르다. Procedure는 `CREATE PROCEDURE`로 생성하며 값을 반환하지 않는다(PostgreSQL 11에서 도입).

### 6.2 EXCLUSION Constraint

고유한 pending 주문을 보장하기 위한 고급 constraint:

```sql
ALTER TABLE sales.orders
ADD CONSTRAINT one_pending_order_per_customer
EXCLUDE USING btree (customer_id WITH =)
WHERE (status = 'pending');
```

이는 "같은 `customer_id`를 가지면서 동시에 `status = 'pending'`인 행이 2개 이상 존재할 수 없다"는 규칙이다. MySQL에는 없는 PostgreSQL 고유 기능이다.

### 6.3 Trigger

특정 테이블에 DML이 발생할 때 자동으로 실행되는 함수이다.

```sql
-- 1. Trigger Function 정의
CREATE OR REPLACE FUNCTION sales.update_order_total()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE sales.orders
    SET total_amount = (
        SELECT COALESCE(SUM(oi.quantity * oi.price), 0)
        FROM sales.order_items oi
        WHERE oi.order_id = COALESCE(NEW.order_id, OLD.order_id)
    )
    WHERE id = COALESCE(NEW.order_id, OLD.order_id)
    AND status = 'pending';
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 2. Trigger 등록
CREATE TRIGGER trigger_update_order_total
AFTER INSERT OR UPDATE OR DELETE
ON sales.order_items
FOR EACH ROW
EXECUTE FUNCTION sales.update_order_total();
```

```
+-------------------+     INSERT/UPDATE/DELETE     +-------------------+
| sales.order_items | ---------------------------> | TRIGGER fires     |
+-------------------+                              |                   |
                                                   | update_order_total|
                                                   |   -> recalc SUM   |
                                                   |   -> UPDATE orders |
                                                   +-------------------+
```

**Trigger 내 특수 변수**:

| 변수 | INSERT | UPDATE | DELETE |
|------|--------|--------|--------|
| `NEW` | 새 행 | 변경된 행 | NULL |
| `OLD` | NULL | 변경 전 행 | 삭제된 행 |

**추가 팁**: LISTEN/NOTIFY를 사용하면 데이터 변경 시 비동기 알림을 클라이언트에 전송할 수 있다 (메시지 큐 패턴).

---

## 7. View & Materialized View

### 7.1 일반 View

호출할 때마다 내부 쿼리가 실행된다 (항상 최신 데이터).

```sql
CREATE VIEW sales.product_sales_summary AS
SELECT c.name AS product_name, c.category,
       SUM(oi.quantity) AS total_quantity_sold,
       SUM(oi.quantity * oi.price) AS total_revenue
FROM products.catalog c
LEFT JOIN sales.order_items oi ON c.id = oi.product_id
GROUP BY c.id
ORDER BY total_quantity_sold DESC, total_revenue DESC;

-- 사용
SELECT * FROM sales.product_sales_summary WHERE category = 'coffee';
```

### 7.2 Materialized View

쿼리 결과를 물리적으로 저장(캐싱)한다. 명시적 REFRESH 전까지 동일한 결과를 반환한다.

```sql
CREATE MATERIALIZED VIEW sales.monthly_sales_summary AS
SELECT date_trunc('month', o.order_date) AS sales_month,
       SUM(oi.quantity * oi.price) AS total_revenue,
       COUNT(DISTINCT(o.id)) AS total_orders
FROM sales.orders o
JOIN sales.order_items oi ON o.id = oi.order_id
GROUP BY sales_month
ORDER BY sales_month;

-- 데이터 갱신
REFRESH MATERIALIZED VIEW sales.monthly_sales_summary;
```

```
+------------------+          +------------------------+
|   Regular View   |          |  Materialized View     |
+------------------+          +------------------------+
| query on every   |          | query once, cache      |
| SELECT call      |          | result on disk         |
|                  |          |                        |
| always fresh     |          | stale until REFRESH    |
| higher cost per  |          | lower cost per query   |
| query            |          | higher storage cost    |
+------------------+          +------------------------+
```

주기적 갱신에는 `pg_cron` 확장 또는 Trigger를 활용할 수 있다.

---

## 8. Role & Access Control

PostgreSQL에서 "사용자"는 `LOGIN` 속성을 가진 **Role**이다.

### 8.1 Role 생성 및 권한 설정

```sql
-- 1. Role 생성 (LOGIN 가능)
CREATE ROLE coffee_chain_admin WITH LOGIN PASSWORD 'password';

-- 2. 특정 DB 접속 허용
GRANT CONNECT ON DATABASE coffee_chain TO coffee_chain_admin;

-- 3. 다른 모든 Role의 접속 차단
REVOKE CONNECT ON DATABASE coffee_chain FROM PUBLIC;
REVOKE CONNECT ON DATABASE brewery FROM PUBLIC;
REVOKE CONNECT ON DATABASE postgres FROM PUBLIC;

-- 4. Schema 사용 권한
GRANT USAGE ON SCHEMA products TO coffee_chain_admin;
GRANT USAGE ON SCHEMA customers TO coffee_chain_admin;
GRANT USAGE ON SCHEMA sales TO coffee_chain_admin;

-- 5. 테이블 DML 권한
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA products TO coffee_chain_admin;

-- 6. Sequence 사용 권한 (ID 자동 생성에 필요)
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA products TO coffee_chain_admin;
```

**PUBLIC Role**: PostgreSQL의 기본 그룹 Role로, 모든 Role을 포함한다. `REVOKE ... FROM PUBLIC`은 명시적으로 `GRANT`받은 Role에는 영향을 주지 않는다.

### 8.2 권한 검증

```
+-----------------------------+
|    coffee_chain_admin       |
+-----------------------------+
| CONNECT coffee_chain    [O] |
| CONNECT brewery         [X] |
| CONNECT postgres        [X] |
| SELECT/INSERT/UPDATE/   [O] |
|   DELETE on tables          |
| CREATE TABLE            [X] |
| DROP TABLE              [X] |
+-----------------------------+
  (owner가 아니므로 DDL 불가)
```

`postgres` Role(Superuser)은 DB 구조 초기화/변경이 필요한 관리 컴포넌트에서만 제한적으로 사용해야 한다.

---

## 멀티테넌시 전략 비교

```
+---------------------+------------------+------------------+------------------+
|                     | Table-level      | Schema-level     | Database-level   |
+---------------------+------------------+------------------+------------------+
| Isolation method    | tenant_id column | separate schema  | separate DB      |
|                     |                  | per tenant       | per tenant       |
+---------------------+------------------+------------------+------------------+
| Cross-tenant query  | easy (same tbl)  | easy (same DB)   | not possible     |
|                     |                  |                  | (without ext)    |
+---------------------+------------------+------------------+------------------+
| Data isolation      | low              | medium           | high             |
+---------------------+------------------+------------------+------------------+
| Scalability         | vertical only    | vertical only    | horizontal       |
|                     |                  |                  | (DB per server)  |
+---------------------+------------------+------------------+------------------+
| Complexity          | low              | medium           | high             |
+---------------------+------------------+------------------+------------------+
```

---

## Quiz

**Q1.** PostgreSQL에서 서로 다른 Schema에 있는 테이블 간 JOIN은 가능한가? 서로 다른 Database에 있는 테이블 간 JOIN은 어떠한가?

<details>
<summary>정답</summary>
같은 Database 내의 서로 다른 Schema 간 JOIN은 가능하다. 서로 다른 Database 간의 cross-database JOIN은 기본적으로 불가능하며, 서드파티 확장(예: dblink, postgres_fdw)이 필요하다.
</details>

**Q2.** `SERIAL`, `BIGSERIAL`, `UUID` 중 분산 환경에서 Sequence 의존성을 제거하고 싶을 때 적합한 타입은? 또한 PostgreSQL 18에서 도입된 `uuidv7()`이 기존 `gen_random_uuid()`(UUIDv4)보다 인덱스 성능에 유리한 이유는 무엇인가?

<details>
<summary>정답</summary>
UUID 타입이 적합하다. UUIDv7은 Unix timestamp를 포함하므로, 시간적으로 가까운 시점에 생성된 UUID가 인덱스에서 인접하게 저장된다. 이로 인해 B-tree 인덱스 탐색 시 캐시 효율이 높아지고 삽입 성능이 향상된다. 반면 UUIDv4는 완전 랜덤이므로 인덱스 전체에 분산 삽입되어 성능이 떨어질 수 있다.
</details>

**Q3.** `ALTER TABLE`로 CHECK constraint를 추가할 때, 기존 데이터에 해당 constraint를 위반하는 행이 있으면 어떤 일이 발생하는가? 이 문제를 해결하는 절차는?

<details>
<summary>정답</summary>
ALTER TABLE 문이 에러와 함께 실패한다 (예: "check constraint ... is violated by some row"). 해결 절차: (1) 위반 데이터를 먼저 UPDATE하여 constraint 조건에 맞도록 수정한 후, (2) ALTER TABLE로 constraint를 추가한다.
</details>

**Q4.** 두 세션이 동일한 행에 대해 동시에 UPDATE를 실행할 때, PostgreSQL의 기본 격리 수준(Read Committed)에서의 동작을 설명하라. Dirty Read와 Dirty Write가 각각 어떻게 방지되는가?

<details>
<summary>정답</summary>
Read Committed에서 SELECT는 다른 트랜잭션의 미커밋 변경을 볼 수 없다(Dirty Read 방지). 두 트랜잭션이 같은 행을 UPDATE하려 하면, 먼저 잠금을 획득한 트랜잭션이 완료될 때까지 나중 트랜잭션은 BLOCK된다. 나중 트랜잭션이 unblock되면 커밋된 최신 값을 기반으로 UPDATE를 수행한다(Dirty Write 방지). ROLLBACK된 데이터는 MVCC에 의해 보이지 않게 되며, 물리적 제거는 VACUUM이 담당한다.
</details>

**Q5.** PostgreSQL의 EXCLUSION constraint는 무엇이며, MySQL에 동등한 기능이 있는가? 아래 구문이 보장하는 규칙을 설명하라.
```sql
EXCLUDE USING btree (customer_id WITH =) WHERE (status = 'pending')
```

<details>
<summary>정답</summary>
EXCLUSION constraint는 지정된 컬럼/연산자 조합으로 두 행을 비교했을 때 모두 true가 되는 경우를 금지한다. 위 구문은 "같은 customer_id를 가지면서 status = 'pending'인 행이 2개 이상 동시에 존재할 수 없다"는 규칙을 보장한다. 즉 고객당 pending 주문은 최대 1개만 허용된다. MySQL에는 EXCLUSION constraint에 해당하는 기능이 없으며, 애플리케이션 레벨 또는 트리거로 유사한 로직을 구현해야 한다.
</details>

**Q6.** 일반 View와 Materialized View의 차이를 설명하고, Materialized View의 데이터가 최신이 아닌 상태(stale)가 되었을 때 갱신하는 명령어는 무엇인가?

<details>
<summary>정답</summary>
일반 View는 호출할 때마다 내부 쿼리를 실행하므로 항상 최신 데이터를 반환하지만 매번 쿼리 비용이 발생한다. Materialized View는 쿼리 결과를 물리적으로 저장(캐싱)하여 읽기 성능이 빠르지만, 원본 데이터가 변경되어도 자동으로 갱신되지 않는다. 갱신 명령어는 `REFRESH MATERIALIZED VIEW view_name;`이다. 주기적 자동 갱신에는 pg_cron 확장이나 트리거를 활용할 수 있다.
</details>