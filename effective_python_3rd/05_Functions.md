# 05. Functions

## 챕터 개요 (3줄 요약)
- 함수는 파이썬의 첫 번째 조직화 도구로, 가독성·재사용·리팩터링을 돕는 다양한 고유 기능을 제공한다.
- 인자 전달(참조 전달, 키워드/위치 전용, 가변 인자)과 반환(예외 vs None, 결과 객체)을 명확히 설계해야 버그를 줄인다.
- 클로저, 데코레이터, functools.partial 같은 함수형 도구로 인터페이스를 깔끔하게 연결할 수 있다.

---

## Item 30: 함수 인자가 변경될 수 있음을 알라 (Know That Function Arguments Can Be Mutated)
> 파이썬 인자는 참조로 전달되어, 받은 함수가 가변 객체를 수정할 수 있다.

- 정수·문자열 같은 불변 객체는 값 전달처럼 보이지만, 리스트·딕트·객체는 수정 가능하다.
- 변수 할당은 별칭(alias)을 만들어 원본을 함께 변경할 수 있다.
- 보호하려면 슬라이스 `[:]`나 `dict.copy()`로 방어적 복사본을 만든다.
- 인자를 수정한다면 함수명/인자명/문서로 명시하고, 그 외엔 수정하지 않는다.
- 함수형 스타일과 불변 객체(dataclass)를 활용하는 것도 좋은 대안이다.

```python
def capitalize_items(items):
    for i in range(len(items)):
        items[i] = items[i].capitalize()

my_items = ["hello", "world"]
capitalize_items(my_items[:])   # 복사본 전달로 원본 보호
```

## Item 31: 3개 초과 변수 언패킹 대신 전용 결과 객체를 반환하라 (Return Dedicated Result Objects)
> 함수가 4개 이상의 값을 반환하면 순서 혼동 등 오류가 잦으므로 경량 클래스를 쓴다.

- 튜플 반환과 언패킹으로 여러 값을 반환할 수 있다.
- 별표 표현식으로 캐치올 언패킹도 가능하다.
- 값이 많으면 순서를 헷갈려 미묘한 버그가 생긴다.
- 언패킹은 3개 변수 이하로 제한한다.
- 그 이상이면 `dataclass` 같은 경량 클래스를 반환한다.

```python
from dataclasses import dataclass

@dataclass
class Stats:
    minimum: float
    maximum: float
    average: float
    median: float
    count: int
```

## Item 32: None 반환보다 예외 발생을 선호하라 (Prefer Raising Exceptions to Returning None)
> 특별한 의미로 None을 반환하면 0·빈 문자열 등도 False라 오해를 부른다.

- None을 에러 표시로 쓰면 `if not result`에서 0 같은 값도 에러로 오인된다.
- (성공여부, 결과) 튜플 반환도 첫 요소를 무시하면 위험하다.
- 더 나은 방법은 특수 상황에 예외를 발생시키는 것이다.
- 호출자는 try/except/else로 결과를 안전하게 사용한다.
- 타입 어노테이션과 docstring으로 반환·예외 동작을 명시한다.

```python
def careful_divide(a: float, b: float) -> float:
    """Divides a by b.
    Raises:
        ValueError: When the inputs cannot be divided.
    """
    try:
        return a / b
    except ZeroDivisionError:
        raise ValueError("Invalid inputs")
```

## Item 33: 클로저와 변수 스코프, nonlocal의 상호작용을 알라 (Know How Closures Interact with Variable Scope and nonlocal)
> 클로저는 외부 스코프 변수를 참조할 수 있지만, 할당은 기본적으로 새 지역 변수를 만든다.

- 클로저는 정의된 외부 스코프의 변수를 참조할 수 있다.
- 변수 참조는 지역→외부→전역→내장 순으로 탐색된다.
- 클로저 내 할당은 기본적으로 지역 변수 정의로 처리된다(스코핑 버그).
- `nonlocal`로 외부 스코프 변수에 할당할 수 있다(모듈 스코프까지는 못 감).
- nonlocal이 복잡해지면 상태를 헬퍼 클래스로 감싸는 것이 낫다.

```python
class Sorter:
    def __init__(self, group):
        self.group = group
        self.found = False
    def __call__(self, x):
        if x in self.group:
            self.found = True
            return (0, x)
        return (1, x)
```

## Item 34: 가변 위치 인자로 시각적 잡음을 줄여라 (Reduce Visual Noise with Variable Positional Arguments)
> *args를 쓰면 빈 리스트를 넘길 필요 없이 호출이 깔끔해진다.

- 마지막 위치 파라미터에 `*`를 붙이면 가변 개수의 인자를 받는다(varargs).
- 시퀀스 앞에 `*`를 붙여 위치 인자로 펼칠 수 있다.
- 제너레이터에 `*`를 쓰면 끝까지 소비되어 메모리 위험이 있다.
- *args는 입력 개수가 작을 때 적합하다.
- *args 함수에 위치 인자를 앞에 추가하면 기존 호출자가 깨질 수 있다(키워드 전용 인자로 확장 권장).

```python
def log(message, *values):
    if not values:
        print(message)
    else:
        print(f"{message}: {', '.join(map(str, values))}")

log("Hi there")           # 빈 리스트 불필요
log("Numbers", 1, 2, 3)
```

## Item 35: 키워드 인자로 선택적 동작을 제공하라 (Provide Optional Behavior with Keyword Arguments)
> 키워드 인자는 호출의 의도를 명확히 하고, 기본값으로 하위 호환 확장을 가능케 한다.

- 인자는 위치 또는 키워드로 전달할 수 있고, 키워드는 순서가 자유롭다.
- `**` 연산자로 딕셔너리를 키워드 인자로 펼친다.
- `**kwargs`로 임의의 키워드 인자를 받을 수 있다.
- 키워드는 각 인자의 목적을 명확히 한다.
- 기본값 키워드로 기존 호출자를 깨지 않고 새 동작을 추가한다(선택 인자는 항상 키워드로 전달).

```python
def flow_rate(weight_diff, time_diff, period=1, units_per_kg=1):
    return ((weight_diff * units_per_kg) / time_diff) * period

flow = flow_rate(0.5, 3, period=3600, units_per_kg=2.2)
```

## Item 36: 동적 기본 인자엔 None과 docstring을 사용하라 (Use None and Docstrings to Specify Dynamic Default Arguments)
> 기본 인자값은 모듈 로드 시 한 번만 평가되므로, 동적 기본값엔 None을 쓴다.

- `when=datetime.now()` 같은 기본값은 정의 시점에 한 번만 평가된다.
- 가변 기본값(빈 리스트/딕트)을 쓰면 모든 호출이 같은 객체를 공유한다.
- 기본값을 `None`으로 두고 docstring에 실제 기본 동작을 문서화한다.
- 함수 본문에서 `if x is None`을 검사해 동적 기본값을 생성한다.
- 타입 어노테이션(`datetime | None = None`)과도 잘 동작한다.

```python
def log(message, when=None):
    """Log a message with a timestamp.
    Args:
        when: Defaults to the present time.
    """
    if when is None:
        when = datetime.now()
    print(f"{when}: {message}")
```

## Item 37: 키워드 전용/위치 전용 인자로 명확성을 강제하라 (Enforce Clarity with Keyword-Only and Positional-Only Arguments)
> 키워드 전용·위치 전용 인자로 호출 방식을 강제해 가독성과 결합도를 개선한다.

- `*` 뒤의 인자는 키워드로만 전달 가능(키워드 전용)하여 의도를 명확히 한다.
- `/` 앞의 인자는 위치로만 전달 가능(위치 전용, 3.8+)하여 파라미터명 변경의 자유를 준다.
- 키워드 전용은 불리언 플래그 혼동 같은 버그를 막는다.
- 위치 전용은 내부 파라미터명을 인터페이스에서 분리한다.
- `/`와 `*` 사이 인자는 위치/키워드 둘 다 허용(기본 동작)된다.

```python
def safe_division(numerator, denominator, /, ndigits=10, *,
                  ignore_overflow=False, ignore_zero_division=False):
    try:
        return round(numerator / denominator, ndigits)
    except ZeroDivisionError:
        if ignore_zero_division:
            return float("inf")
        raise
```

## Item 38: functools.wraps로 함수 데코레이터를 정의하라 (Define Function Decorators with functools.wraps)
> 데코레이터는 메타데이터를 잃기 쉬우므로 functools.wraps로 보존해야 한다.

- 데코레이터는 함수 호출 전후에 코드를 실행해 인자·반환·예외를 다룬다.
- `@` 문법은 `func = decorator(func)`와 동일하다.
- 순진한 래퍼는 `__name__`, docstring 등이 wrapper로 바뀌어 디버거·help·pickle을 망친다.
- `functools.wraps`를 wrapper에 적용하면 원본 메타데이터를 복사한다.
- `__name__`, `__module__`, `__annotations__` 등 인터페이스가 보존된다.

```python
from functools import wraps

def trace(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)
        print(f"{func.__name__}({args!r}, {kwargs!r}) -> {result!r}")
        return result
    return wrapper
```

## Item 39: 글루 함수엔 lambda보다 functools.partial을 선호하라 (Prefer functools.partial over lambda)
> 인자를 고정하거나 재배치할 때 partial이 lambda보다 명료하고 디버깅에 유리하다.

- 많은 API가 단순 함수를 받지만 시그니처가 안 맞을 때가 있다.
- lambda로 인자를 재배치하거나 일부 값을 고정할 수 있다(일회성엔 적합).
- 일부 인자를 고정하는 부분 적용(partial application/currying)이 흔하다.
- `functools.partial`은 위치·키워드 인자를 고정하며 `.args`, `.keywords`, `.func`로 검사 가능하다.
- 인자 순서를 바꿔야 할 때만 lambda를 쓴다.

```python
import functools, math

log_sum_e = functools.partial(logn_sum_last, base=math.e)
print(log_sum_e.keywords)   # {'base': 2.718...}
```

---

## Summary (핵심 정리)
- 인자는 참조로 전달되니 가변 객체 수정에 주의하고, 동적 기본값엔 None+docstring을 사용한다.
- 키워드/위치 전용 인자와 *args/**kwargs로 명확하고 하위 호환되는 인터페이스를 설계한다.
- 반환은 None보다 예외를, 다수 값은 결과 객체를 쓰고, 데코레이터엔 functools.wraps, 글루엔 partial을 활용한다.
