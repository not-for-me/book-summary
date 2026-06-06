# PostgreSQL의 JSON 데이터 처리

- **키워드**: `jsonb`, `json`, GIN 인덱스, JSON Path Expression, `@>`, `?`, `->`, `->>`, `jsonb_set`, `#-`, `jsonb_path_ops`
- **출처**: [Just Use Postgres! - Chapter 5: Postgres and JSON](https://learning.oreilly.com/library/view/just-use-postgres/9781633435698/Text/chapter-5.html)
- **3줄 요약**
  1. PostgreSQL은 `json`과 `jsonb` 두 가지 JSON 전용 데이터 타입을 제공하며, 바이너리 포맷으로 저장하여 검색 성능이 우수한 `jsonb`가 기본 권장 타입이다.
  2. `->`, `->>`, `@>`, `?` 연산자와 SQL/JSON Path 표현식을 활용하면 중첩된 JSON 구조를 효율적으로 조회·필터링·수정할 수 있다.
  3. JSON 데이터에는 기본 GIN 인덱스(`jsonb_ops`)와 경로 기반 GIN 인덱스(`jsonb_path_ops`)를 사용할 수 있으며, 각각 지원하는 연산자와 성능 특성이 다르다.

---

## 1. JSON 데이터 저장: text vs json vs jsonb

MySQL에서는 5.7부터 JSON 타입을 지원하지만, PostgreSQL은 2012년부터 JSON을 지원하며 훨씬 풍부한 연산자·함수·인덱스 체계를 갖추고 있다.

PostgreSQL에서 JSON 데이터를 저장하는 3가지 방식의 비교:

```
+------------+------------------+------------------+--------------------+
|            |      text        |      json        |      jsonb         |
+------------+------------------+------------------+--------------------+
| Storage    | Plain text       | Plain text       | Binary format      |
| Write      | Fast             | Fast             | Slightly slower    |
| Read/Query | No JSON ops      | Parse every read | Fast (pre-parsed)  |
| Key order  | Preserved        | Preserved        | May reorder        |
| Whitespace | Preserved        | Preserved        | Removed            |
| Dup keys   | Preserved        | Preserved        | Removed            |
| GIN Index  | Not supported    | Not supported    | Supported          |
| Operators  | None             | Limited          | Full set           |
+------------+------------------+------------------+--------------------+
```

**핵심**: 특별히 키 순서를 보존해야 하는 요구사항이 없다면 `jsonb`를 기본으로 사용한다. `jsonb`는 쓰기 시 파싱·변환 비용이 있지만, 읽기 시 매번 파싱하지 않아도 되고 GIN 인덱스를 통한 고속 검색이 가능하다.

---

## 2. 하이브리드 모델: JSON과 관계형 타입의 균형

JSON을 사용해야 하는 경우:
- 정적이거나 드물게 변경되는 데이터 (설정, 메타데이터, 고객 선호도)
- 희소(sparse) 데이터 (대부분 null인 선택적 필드가 많은 경우)
- 스키마 유연성이 필요하거나 정규화가 어려운 경우 (외부 API 응답, 텔레메트리 이벤트)

관계형 타입을 사용해야 하는 경우:
- 빈번한 UPDATE, JOIN, 집계, 필터링 등에서 성능이 중요한 경우

하이브리드 접근법의 예시 테이블:

```sql
pizzeria.order_items (
    order_id      INT NOT NULL,          -- Relational
    order_item_id INT NOT NULL,          -- Relational
    pizza         JSONB NOT NULL,        -- JSON (recipe)
    price         NUMERIC(5,2) NOT NULL, -- Relational
    PRIMARY KEY (order_id, order_item_id)
);
```

자주 조회·조인·제약조건이 필요한 컬럼(`order_id`, `price`)은 관계형 타입으로, 복잡하고 정적인 데이터(`pizza` 레시피)는 `jsonb`로 저장한다.

---

## 3. JSON 조회 연산자 및 함수

### 3.1 `->` 와 `->>` 연산자

```
+----------+------------------+------------------+
|          |       ->         |       ->>        |
+----------+------------------+------------------+
| Returns  | JSON type        | TEXT type        |
| Text val | "small" (quoted) | small (unquoted) |
| Use case | Chaining         | Final extraction |
+----------+------------------+------------------+
```

체이닝 예시 — 중첩된 JSON 배열 접근:

```sql
-- veggies 배열 전체 조회
pizza->'toppings'->'veggies'

-- 배열의 첫 번째 요소(index 0) 조회
pizza->'toppings'->'veggies'->0

-- 첫 번째 veggie의 onion 값을 TEXT로 추출
pizza->'toppings'->'veggies'->0->>'onion'
```

`WHERE` 절에서 사용 시 주의: `->` 연산자는 JSON 포맷을 반환하므로 비교값에 쌍따옴표(`'"small"'`)를 포함해야 하고, `->>` 연산자는 일반 텍스트(`'small'`)와 비교한다. 이를 혼동하면 `Token "small" is invalid` 에러가 발생한다.

### 3.2 `?` 연산자 (키 존재 여부 확인, jsonb 전용)

특정 키가 JSON 객체에 존재하는지 확인한다.

```sql
-- toppings 안에 'meats' 키가 존재하는 주문만 조회
WHERE pizza->'toppings' ? 'meats'
```

배열 내부 객체의 키를 확인하려면 `jsonb_array_elements`와 서브쿼리를 조합한다:

```sql
WHERE EXISTS (
    SELECT 1
    FROM jsonb_array_elements(pizza->'toppings'->'meats') AS meats
    WHERE meats ? 'sausage'
)
```

### 3.3 `@>` 연산자 (포함 관계 확인, jsonb 전용)

좌측 JSON 객체가 우측 JSON 객체를 포함하는지 검사한다.

```sql
-- 단일 필드 포함
WHERE pizza @> '{"crust": "gluten_free"}'

-- 복수 필드 포함
WHERE pizza @> '{"crust": "gluten_free", "type": "custom"}'

-- 중첩 구조 포함
WHERE pizza @> '{"toppings": {"veggies": [{"tomato": "extra"}]}}'
```

`->` 와 `@>` 를 조합하여 가독성을 높일 수 있다:

```sql
WHERE pizza @> '{"crust": "gluten_free", "type": "custom"}'
  AND pizza->'toppings'->'veggies' @> '[{"tomato": "extra"}]'
```

### 3.4 JSON Path 표현식

SQL/JSON Path 언어를 사용하여 복잡한 JSON 탐색을 간결하게 표현한다.

핵심 구문 요소:

```
+----------+-------------------------------------------+
| Symbol   | Description                               |
+----------+-------------------------------------------+
| $        | Root JSON object                          |
| .key     | Access field by key name                  |
| [*]      | All elements in array                     |
| [n]      | Element at index n                        |
| ?()      | Filter expression                         |
| @        | Current element in filter                 |
| exists() | Check field existence in filter            |
+----------+-------------------------------------------+
```

주요 함수:

```
+-------------------------+-----------------------------------------------+
| Function                | Description                                   |
+-------------------------+-----------------------------------------------+
| jsonb_path_query()      | Path expression result set return              |
| jsonb_path_exists()     | Path expression match true/false               |
| jsonb_object_keys()     | Extract keys from JSON object                  |
+-------------------------+-----------------------------------------------+
```

사용 예시:

```sql
-- 필드 접근
jsonb_path_query(pizza, '$.type')

-- 배열 전체 요소 접근
jsonb_path_query(pizza, '$.toppings.cheese[*]')

-- 필터: parmesan이 존재하는 주문 확인
jsonb_path_exists(pizza, '$.toppings.cheese[*] ? (exists(@.parmesan))')

-- 필터 체이닝: custom 타입이면서 parmesan이 extra인 주문
jsonb_path_exists(
    pizza,
    '$ ? (@.type == "custom") .toppings.cheese[*].parmesan ? (@ == "extra")'
)
```

Path 표현식의 실행 흐름(체이닝 필터 예시):

```
  $ (root pizza object)
  |
  v
  ? (@.type == "custom")  ---[false]--> skip row
  |
  [true]
  v
  .toppings.cheese[*].parmesan
  |
  v
  ? (@ == "extra")  ---[false]--> skip row
  |
  [true]
  v
  Include in result
```

---

## 4. JSON 데이터 수정

### 4.1 `jsonb_set` 함수

```
jsonb_set(target, path, new_value, create_if_missing)
```

```
+---------------------+--------------------------------------------+
| Argument            | Description                                |
+---------------------+--------------------------------------------+
| target              | Original JSONB object                      |
| path                | '{key1, key2, ...}' to target field        |
| new_value           | New JSON value (must be valid JSON)        |
| create_if_missing   | true: add if absent / false: update only   |
+---------------------+--------------------------------------------+
```

```sql
-- 단일 필드 수정: crust를 regular로 변경
UPDATE pizzeria.order_items
SET pizza = jsonb_set(pizza, '{crust}', '"regular"', false)
WHERE order_id = 20 AND order_item_id = 5;

-- 배열 전체 교체: veggies 토핑 변경
UPDATE pizzeria.order_items
SET pizza = jsonb_set(
    pizza, '{toppings,veggies}',
    '[{"tomato":"extra"}, {"spinach":"regular"}]', false
) WHERE order_id = 20 AND order_item_id = 5;
```

배열 내 특정 인덱스 요소만 수정하려면 경로에 인덱스를 포함한다: `'{toppings, veggies, 0, tomato}'`

### 4.2 `#-` 연산자 (필드 삭제)

```sql
-- meats 토핑 전체 제거
UPDATE pizzeria.order_items
SET pizza = pizza #- '{toppings,meats}'
WHERE order_id = 20 AND order_item_id = 5;
```

---

## 5. JSON 인덱싱

### 5.1 B-tree Expression Index

`->>` 연산자로 추출한 특정 필드에 대해 B-tree 인덱스를 생성한다.

```sql
CREATE INDEX idx_pizza_type
ON pizzeria.order_items ((pizza ->> 'type'));
```

장점: 특정 필드 조회가 빠르다.
단점: 인덱싱한 표현식과 **정확히 일치하는** 쿼리만 인덱스를 사용한다. `pizza ->> 'type'`으로 만든 인덱스는 `pizza -> 'type'`을 사용하는 쿼리에서 무시된다. 필드마다 별도 인덱스가 필요하여 확장성이 떨어진다.

### 5.2 GIN 인덱스 (기본: `jsonb_ops`)

```sql
CREATE INDEX idx_pizza_orders_gin
ON pizzeria.order_items USING GIN(pizza);
```

JSON 객체의 모든 키·값·배열 요소를 개별 항목으로 추출하여 인덱스에 저장한다.

```
  Sample pizza JSON:
  {"size":"large", "type":"three cheese", "toppings":{"cheese":[...]}}

          Default GIN Index Structure
  +----------------------------------------------------+
  |              Root Index Page                        |
  |  [extra] --- [marinara] --- [...]                   |
  +------/-------------\----------------\---------------+
        v               v                v
  +-----------+   +-----------+    +-----------+
  | cheese    |   | marinara  |    | size      |
  |  -> tr1   |   |  -> tr1   |    |  -> tr1   |
  |  -> tr2   |   |  -> tr5   |    |  -> tr3   |
  | cheddar   |   | mozzarella|    | thin      |
  |  -> tr1   |   |  -> tr1   |    |  -> tr1   |
  |  -> tr4   |   |  -> tr2   |    |  -> tr7   |
  +-----------+   +-----------+    +-----------+
```

지원 연산자: `?`, `@>`, `@?`, `@@`

검색 과정:
1. `WHERE` 조건에서 키·값을 추출
2. GIN 인덱스에서 해당 항목을 찾아 Bitmap 생성 (Bitmap Index Scan)
3. Bitmap에 등록된 테이블 행을 방문하여 실제 조건과 정확히 일치하는지 재검증 (Bitmap Heap Scan)

### 5.3 GIN 인덱스 (`jsonb_path_ops`)

```sql
CREATE INDEX idx_pizza_orders_paths_ops_gin
ON pizzeria.order_items USING GIN (pizza jsonb_path_ops);
```

루트부터 각 값까지의 전체 경로를 해시 코드로 변환하여 저장한다.

```
  Sample paths hashed:
  hash(size.large)
  hash(type.three cheese)
  hash(crust.thin)
  hash(toppings.cheese.cheddar.regular)
  ...

      jsonb_path_ops GIN Index Structure
  +----------------------------------------------------+
  |              Root Index Page                        |
  | [hash(crust.thin)] - [hash(sauce.alfredo)] - [...] |
  +------/-------------------\--------------\----------+
        v                     v              v
  +----------------+  +----------------+  +----------------+
  | hash(crust     |  | hash(sauce     |  | hash(toppings  |
  |  .thin)        |  |  .marinara)    |  |  .cheese       |
  |  -> tr1, tr5   |  |  -> tr1        |  |  .cheddar      |
  | hash(size      |  | hash(size      |  |  .regular)     |
  |  .large)       |  |  .small)       |  |  -> tr1, tr4   |
  |  -> tr1, tr3   |  |  -> tr2, tr6   |  |                |
  +----------------+  +----------------+  +----------------+
```

두 GIN 인덱스 비교:

```
+--------------------+----------------------+------------------------+
|                    | Default (jsonb_ops)  | jsonb_path_ops         |
+--------------------+----------------------+------------------------+
| Index contents     | Distinct keys+values | Hashed root-to-value   |
|                    |                      | paths                  |
+--------------------+----------------------+------------------------+
| Storage size       | Larger               | ~50% smaller           |
+--------------------+----------------------+------------------------+
| Supported ops      | ?, @>, @?, @@        | @>, @?, @@             |
|                    |                      | (NO ? operator)        |
+--------------------+----------------------+------------------------+
| Lookup method      | Text comparison      | Hash comparison        |
|                    |                      | (faster)               |
+--------------------+----------------------+------------------------+
| Best for           | Key existence checks | Containment queries    |
|                    | + containment        | with known paths       |
+--------------------+----------------------+------------------------+
```

**실무 가이드**: 대부분의 쿼리가 `@>` 포함 연산자를 사용한다면 `jsonb_path_ops`가 더 작고 빠르다. `?` 연산자로 키 존재 여부를 확인해야 한다면 기본 GIN 인덱스가 필요하다. 두 인덱스를 동시에 생성할 수도 있으며, PostgreSQL 옵티마이저가 쿼리에 따라 적절한 인덱스를 자동으로 선택한다.

---

## Quiz

**Q1.** PostgreSQL에서 JSON 데이터를 저장할 때 `json` 타입 대신 `jsonb` 타입을 기본으로 권장하는 이유 2가지를 설명하시오.

**Q2.** 다음 두 쿼리의 결과 차이를 설명하시오.
```sql
-- A
SELECT pizza->'size' FROM pizzeria.order_items WHERE order_id = 1;
-- B
SELECT pizza->>'size' FROM pizzeria.order_items WHERE order_id = 1;
```

**Q3.** `@>` 연산자를 사용하여 "thin 크러스트이면서 marinara 소스인 주문"을 조회하는 `WHERE` 절을 작성하시오.

**Q4.** 다음 JSON Path 표현식이 하는 일을 단계별로 설명하시오.
```sql
jsonb_path_exists(pizza, '$ ? (@.type == "custom") .toppings.cheese[*].parmesan ? (@ == "extra")')
```

**Q5.** B-tree Expression Index(`CREATE INDEX ... ((pizza ->> 'type'))`)의 한계와, 이를 대체할 수 있는 GIN 인덱스의 장점을 비교 설명하시오.

**Q6.** 기본 GIN 인덱스(`jsonb_ops`)와 `jsonb_path_ops` GIN 인덱스의 내부 저장 방식 차이를 설명하고, `?` 연산자가 `jsonb_path_ops` 인덱스에서 지원되지 않는 이유를 서술하시오.