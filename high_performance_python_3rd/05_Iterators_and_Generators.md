# 05. Iterators and Generators

## 챕터 개요 (3줄 요약)

- 제너레이터(generator)는 지연 평가(lazy evaluation)로 한 번에 현재 값 하나만 다뤄, 전체 데이터를 메모리에 미리 만들지 않고도 리스트 수준의 가독성을 유지한다.
- `yield`로 값을 하나씩 내보내며 상태를 유지하므로, 무한 수열이나 메모리에 담을 수 없는 대용량 데이터(single-pass/online 알고리즘)를 처리할 수 있다.
- `itertools`(islice, chain, takewhile, groupby 등)로 제너레이터를 조합하면 데이터 생성과 변환을 분리한 메모리 효율적 파이프라인을 구성할 수 있다.

---

## 1. Generators vs Lists (제너레이터 vs 리스트)

> 리스트 버전은 전체 결과를 미리 만들어 저장하지만, 제너레이터는 `yield`로 값을 하나씩 내보내며 필요할 때마다 이전 상태를 복원해 다음 값을 계산한다.

- `range`처럼 Python의 for 루프는 미리 배열을 만들지 않고 지연 평가로 값을 생성한다(엄밀히 range는 range 타입).
- 리스트 버전 `fibonacci_list`는 num_items만큼 append하며 전체를 저장하지만, 제너레이터 `fibonacci_gen`은 `yield`로 값을 하나씩 내보낸다.
- 제너레이터가 끝에 도달하면 StopIteration 예외가 발생해 더 이상 값이 없음을 알린다.
- 10만 개 피보나치에서 제너레이터는 약 2.7배 빠르고 측정 가능한 메모리를 거의 쓰지 않은 반면, 리스트는 약 418MB를 썼다.
- for 루프는 내부적으로 `iter()`로 이터레이터를 만들고 `next()`를 StopIteration까지 호출하는 것과 같다.
- 단, 같은 데이터를 여러 번 참조해야 하면 제너레이터는 매번 재계산하므로, CPU 속도와 메모리 효율 사이의 트레이드오프를 고려해야 한다.

### for 루프의 분해 (개념)

```python
# for i in obj: do_work(i) 는 다음과 동등
it = iter(obj)
while True:
    try:
        i = next(it)
    except StopIteration:
        break
    else:
        do_work(i)
```

### 리스트 vs 제너레이터 (원문 의도 유지, 새 예제)

```python
# 메모리 점유: 전체 저장
def squares_list(n):
    out = []
    for i in range(n):
        out.append(i * i)
    return out

# 지연 평가: 값 하나씩 생성 (메모리 거의 0)
def squares_gen(n):
    for i in range(n):
        yield i * i
```

---

## 2. Generator Comprehension & Built-ins (제너레이터 컴프리헨션과 내장 함수)

> 대괄호 대신 소괄호를 쓰면 리스트가 아닌 제너레이터가 생성되며, range·map·zip·filter·enumerate 같은 내장 함수도 대부분 제너레이터다.

- `[v for ... if ...]`(리스트)와 `(v for ... if ...)`(제너레이터)의 미묘한 차이로 메모리를 크게 절약할 수 있다.
- 제너레이터는 length 속성이 없으므로, 개수를 셀 때 `sum(1 for ... if ...)` 패턴으로 메모리 없이 계산한다.
- `len([n for n in gen if ...])`는 불필요하게 배열을 만들어 약 98MB를 낭비하므로 `sum(1 for ...)`이 낫다.
- range·map·zip·filter·reversed·enumerate는 전체 결과를 저장하지 않고 필요할 때 계산한다.
- 예: `zip(range(100_000), range(100_000))`은 메모리에 두 숫자만 유지한다.

### 메모리 없이 개수 세기

```python
# 나쁨: 리스트를 만들고 길이만 취한 뒤 버림 (메모리 낭비)
count = len([n for n in numbers if n % 3 == 0])

# 좋음: 제너레이터로 1을 합산 → 메모리 거의 0
count = sum(1 for n in numbers if n % 3 == 0)
```

---

## 3. Iterators for Infinite Series (무한 수열 이터레이터)

> 제너레이터는 무한 수열을 함수로 캡슐화할 수 있어, 원하는 만큼만 값을 취하고 코드가 충분하다고 판단하면 종료할 수 있다.

- `while True: yield ...` 형태로 무한 피보나치 같은 스트림을 만들 수 있다.
- 같은 질문(5000 미만 홀수 피보나치 개수)을 naive·transform·succinct 등 여러 방식으로 풀 수 있다.
- `fibonacci_transform`은 명시적이라 디버깅·이해가 쉽고 일반화(다른 수열에도 적용)가 가능하다.
- itertools를 과하게 쓴 succinct 버전은 간결하지만 un-Pythonic해질 수 있다.
- 핵심은 데이터 "생성"과 "변환"의 두 단계를 분리하는 것으로, 제너레이터가 생성을, 일반 함수가 변환을 담당하면 명료해진다.

### 무한 수열과 변환 분리 (원문 의도 유지, 새 예제)

```python
def naturals():            # 무한 생성기
    n = 0
    while True:
        yield n
        n += 1

from itertools import takewhile
# 생성과 변환 분리: 100 미만 짝수의 합
result = sum(x for x in takewhile(lambda v: v < 100, naturals()) if x % 2 == 0)
```

---

## 4. Lazy Evaluation with itertools (itertools로 지연 평가 파이프라인)

> 제너레이터는 현재 값만 다루는 single-pass/online 방식이라 사용이 까다로울 수 있지만, itertools가 복잡한 워크플로를 단순화한다.

- 주요 itertools 함수: islice(무한 제너레이터 슬라이싱), chain(여러 제너레이터 연결), takewhile(조건부 종료), cycle(유한→무한 반복).
- 20년치 6억여 개 초 단위 데이터처럼 메모리에 못 올리는 데이터도 제너레이터로 한 줄씩 지연 읽기 할 수 있다.
- `groupby`는 순차적으로 연속된 항목만 그룹화하므로, 키 함수로 날짜별 그룹을 만들 수 있다.
- `filterfalse`로 정규분포 검정(normaltest)을 통과하지 못하는 이상치 그룹만 걸러낸다.
- 제너레이터 체인은 처음 5개 이상치를 얻는 데 필요한 만큼만 데이터를 읽어, 조기 종료 시 런타임을 크게 줄인다.
- 슬라이딩 윈도우는 `collections.deque`로 O(1) 양끝 추가/삭제를 활용하면 최적화되지만, in-place 작업이라 이전 뷰가 파괴된다.

### 지연 읽기와 윈도우 (개념)

```python
def read_lazy(filename):
    with open(filename) as fd:
        for line in fd:          # 한 줄씩 지연 읽기 (전체 로드 X)
            ts, value = line.strip().split(",")
            yield (int(ts), float(value))

from collections import deque
from itertools import islice
def sliding_window(data, size):
    win = deque(islice(data, size), maxlen=size)  # O(1) 양끝 연산
    yield tuple(win)
    for item in data:
        win.append(item)          # 왼쪽 자동 제거
        yield tuple(win)
```

---

## Summary (핵심 정리)

- 제너레이터는 지연 평가로 현재 값 하나만 메모리에 두므로, 메모리에 담을 수 없는 대용량·무한 데이터를 리스트보다 빠르고 적은 메모리로 처리할 수 있다.
- 데이터 "생성"(제너레이터)과 "변환"(일반 함수)을 분리하면 코드가 명료해지고 재사용·병렬화에 유리하며, itertools로 복잡한 파이프라인을 간결하게 조합할 수 있다.
- 이터레이터는 Python의 기본 타입이므로 메모리 절감의 1순위 도구이지만, 여러 번 참조가 필요하거나 상태 관리가 복잡한 경우엔 CPU/메모리 트레이드오프를 신중히 따져야 한다.
