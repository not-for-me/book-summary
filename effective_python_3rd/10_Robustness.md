# 10. Robustness

## 챕터 개요 (3줄 요약)
- 견고함(robustness)은 예상치 못한 상황에서도 프로그램이 신뢰성 있게 동작하도록 만드는 것이다.
- try/except/else/finally의 각 블록과 assert/raise를 적절히 구분해 사용하고, with 문으로 자원 정리를 보장한다.
- 예외 변수 소멸, Exception vs BaseException, 제너레이터 자원 관리 같은 함정을 이해해 프로그램을 프로덕션 수준으로 강화한다.

---

## Item 80: try/except/else/finally의 각 블록을 활용하라 (Take Advantage of Each Block)
> 네 블록은 각자 고유한 역할이 있으며 조합해 명확한 예외 처리를 구성한다.

- `finally`는 예외 발생 여부와 무관하게 정리 코드를 실행한다(파일 닫기 등).
- `else`는 try가 예외 없이 성공했을 때 실행되어 try 블록을 최소화한다.
- else는 성공 케이스를 try/except와 시각적으로 구분한다.
- 네 블록을 함께 쓰면 읽기·쓰기·처리·정리를 직관적으로 조율한다.
- open은 try 밖에 두어야 OSError가 finally를 건너뛴다.

```python
def divide_json(path):
    handle = open(path, "r+")
    try:
        op = json.loads(handle.read())
        value = op["numerator"] / op["denominator"]
    except ZeroDivisionError:
        return UNDEFINED
    else:
        handle.seek(0); handle.write(json.dumps(op))
        return value
    finally:
        handle.close()
```

## Item 81: 내부 가정은 assert로, 누락된 기대는 raise로 (assert Internal Assumptions and raise Missed Expectations)
> raise는 호출자가 처리할 인터페이스 오류를, assert는 구현 가정 검증을 담당한다.

- `raise` 예외는 함수 인터페이스의 일부로 문서화·테스트되어야 한다.
- `assert` 예외는 호출자가 잡으면 안 되는 구현 가정 검증이다.
- assert는 조건 실패 시 디버깅 메시지를 만들어 자기 문서화된다.
- 외부 API엔 raise, 내부 코드엔 assert가 적합하다.
- assert가 유용하려면 호출 코드가 AssertionError를 삼키지 말아야 한다.

```python
class Rating:                  # 외부 API: raise
    def rate(self, rating):
        if not (0 < rating <= self.max_rating):
            raise RatingError("Invalid rating")

class RatingInternal:          # 내부: assert
    def rate(self, rating):
        assert 0 < rating <= self.max_rating, f"Invalid {rating=}"
```

## Item 82: 재사용 가능한 try/finally엔 contextlib과 with를 고려하라 (Consider contextlib and with Statements)
> with 문은 try/finally 로직을 재사용하고 시각적 잡음을 줄인다.

- `with`는 락 획득/해제 같은 정리 코드의 반복을 없앤다.
- `contextlib.contextmanager` 데코레이터로 함수를 with에 쓸 수 있다.
- `yield` 지점에서 with 블록 내용이 실행된다.
- yield한 값은 `as` 타깃에 할당되어 컨텍스트와 상호작용한다.
- 상태 생성과 사용을 분리·디커플링한다.

```python
from contextlib import contextmanager

@contextmanager
def log_level(level, name):
    logger = logging.getLogger(name)
    old = logger.getEffectiveLevel()
    logger.setLevel(level)
    try:
        yield logger
    finally:
        logger.setLevel(old)
```

## Item 83: try 블록은 항상 가능한 짧게 만들어라 (Always Make try Blocks as Short as Possible)
> try 블록에 코드가 많으면 의도치 않은 예외까지 잡게 된다.

- 한 try 블록엔 예상 오류의 단일 원천만 두어야 한다.
- 추가 코드는 else 블록이나 별도 try 문으로 옮긴다.
- 그렇지 않으면 어느 함수가 예외를 냈는지 구분할 수 없다.
- 예상치 못한 예외가 콜 스택 위로 전파되어 디버깅 가능해진다.
- 잘못된 예외를 잡아 숨기는 것을 방지한다.

```python
try:
    request = lookup_request(connection)
except RpcError:
    close_connection(connection)
else:
    if is_cached(connection, request):   # try 밖으로 이동
        request = None
```

## Item 84: 예외 변수가 사라짐에 주의하라 (Beware of Exception Variables Disappearing)
> except 블록의 예외 변수는 블록 밖이나 finally에서 접근할 수 없다.

- `except ... as e`의 e는 except 블록 안에서만 접근 가능하다.
- finally 블록에서도 e에 접근하면 NameError가 난다.
- 둘러싼 스코프에서 쓰려면 다른 변수에 할당해야 한다.
- 모든 분기에서 결과 변수를 미리 초기화해야 안전하다.
- 파이썬이 변수를 함수 스코프로 일관되게 처리하지 않는 또 다른 예다.

```python
result = "Unexpected exception"   # 미리 초기화 필수
try:
    raise MyError(123)
except MyError as e:
    result = e
finally:
    print(f"Log {result=}")
```

## Item 85: Exception 클래스를 잡는 것에 주의하라 (Beware of Catching the Exception Class)
> 광범위한 Exception을 잡으면 의도치 않은 오류를 숨겨 진짜 버그를 가린다.

- `except Exception`은 한 부분을 다른 부분과 격리하는 데 유용하다.
- 그러나 NameError 같은 진짜 버그도 일시적 오류로 오인된다.
- 파이썬은 동적이라 함수 누락 등을 런타임에야 잡는다.
- 광범위 핸들러를 쓸 때는 반드시 오류를 출력/로깅해야 한다.
- 오류 격리는 유용하나 문제를 숨기지 않도록 주의한다.

```python
try:
    summary = run_report("my_data.csv")
except Exception as e:
    print("Fail:", type(e), e)   # 가시성 확보 필수
```

## Item 86: Exception과 BaseException의 차이를 이해하라 (Understand the Difference Between Exception and BaseException)
> BaseException이 루트이며, KeyboardInterrupt 등은 Exception 핸들러를 건너뛴다.

- `BaseException`이 예외 트리의 루트이고 Exception의 부모다.
- `KeyboardInterrupt`, `SystemExit`, `GeneratorExit`는 Exception을 상속하지 않는다.
- 이들은 오류 보고가 아닌 동작 메커니즘이라 잡으면 부작용이 생긴다.
- 정리는 try/finally·with로 하면 BaseException도 안전히 처리된다.
- BaseException을 잡아야 할 때는 `raise`로 올바르게 전파한다.

```python
try:
    while True:
        try:
            do_processing(handle)
        except Exception as e:
            print("Error:", type(e), e)
finally:
    handle.close()   # KeyboardInterrupt에도 항상 실행
```

## Item 87: 향상된 예외 보고엔 traceback을 사용하라 (Use traceback for Enhanced Exception Reporting)
> traceback 모듈로 런타임에 스택 트레이스를 추출·가공해 디버깅을 돕는다.

- 미처리 예외는 인터프리터가 보기 좋은 트레이스백을 출력한다.
- 동시성 프로그램에선 트레이스백이 출력되지 않아 디버깅이 어렵다.
- `traceback.print_tb(e.__traceback__)`로 스택을 출력한다.
- `extract_tb`로 파일명·라인·함수명 등 상세 정보를 추출한다.
- JSON 로그 등 원하는 방식으로 예외를 가공할 수 있다.

```python
import traceback
try:
    do_work(request.body)
except BaseException as e:
    traceback.print_tb(e.__traceback__)
    print(repr(e))
```

## Item 88: 트레이스백 명확화를 위해 예외를 명시적으로 연결하라 (Consider Explicitly Chaining Exceptions)
> raise ... from으로 예외 원인을 명시해 트레이스백을 명확하게 만든다.

- except 안에서 예외를 raise하면 `__context__`에 원본 예외가 저장된다.
- `raise X from e`는 `__cause__`를 설정해 원인을 명시한다.
- 명시적 연결은 `__suppress_context__`를 True로 해 자동 체인 출력을 억제한다.
- `raise X from None`으로 원인을 완전히 숨길 수 있다.
- traceback 모듈로 직접 처리 시 __cause__/__context__를 고려해야 한다.

```python
def lookup_explicit(my_key):
    try:
        return my_dict[my_key]
    except KeyError as e:
        try:
            result = contact_server(my_key)
        except ServerMissingKeyError:
            raise MissingError from e   # 원인 명시
```

## Item 89: 자원은 항상 제너레이터에 전달하고 호출자가 밖에서 정리하라 (Always Pass Resources into Generators)
> 제너레이터의 finally는 소진 시에만 실행되므로 자원은 밖에서 관리한다.

- 일반 함수의 finally는 반환 전 실행되지만, 제너레이터는 소진(StopIteration) 후 실행된다.
- 부분 소비된 제너레이터엔 GC가 `GeneratorExit`를 주입해 종료시킨다.
- GeneratorExit 처리 중 다른 예외는 gc가 삼켜 숨긴다.
- 따라서 제너레이터의 finally/예외 핸들러에 의존할 수 없다.
- 파일·뮤텍스 같은 자원은 밖에서 할당해 인자로 전달한다.

```python
def lengths_handle(handle):       # 자원을 인자로 받음
    for i, line in enumerate(handle):
        yield len(line.strip())

with open("my_file.txt") as handle:   # 호출자가 정리
    it = lengths_handle(handle)
```

## Item 90: __debug__를 False로 설정하지 마라 (Never Set __debug__ to False)
> assert는 -O 플래그로 비활성화되며, 이는 프로그램 유효성을 해친다.

- `assert`는 내부적으로 `if __debug__:` 검사로 변환된다.
- `__debug__`는 기본 True이며 런타임에 수정할 수 없다.
- `-O` 플래그만이 __debug__를 False로 만들어 assert를 건너뛴다.
- 많은 프레임워크가 assert 활성화에 의존하므로 비활성화는 위험하다.
- assert는 실패하지 않아도 버그 범위를 좁히는 데 도움이 된다.

```python
# python3 -O 는 assert를 건너뜀 -> 사용 금지
assert n % 2 == 0, f"{n=} not even"
# 성능이 필요하면 프로파일링·다른 언어를 고려
```

## Item 91: 개발 도구가 아니면 exec와 eval을 피하라 (Avoid exec and eval Unless You're Building a Developer Tool)
> 문자열로 임의 코드를 실행하는 eval/exec는 보안 위험이 커 거의 쓰지 않는다.

- `eval`은 표현식 문자열을 평가해 결과를 반환한다.
- `exec`은 코드 블록을 실행하며 None을 반환하고 스코프 딕셔너리로 데이터를 주고받는다.
- 일반 앱 코드에 eval/exec이 있으면 심각한 문제의 신호다.
- 입력 채널과 연결되면 심각한 보안 취약점이 된다.
- 디버거·노트북·REPL 같은 개발 도구에서만 적절하다.

```python
# 보안 위험: 일반 코드에선 피할 것
x = eval("1 + 2")   # 3
# 플러그인은 동적 import 등 더 안전한 방법 사용
```

---

## Summary (핵심 정리)
- try/except/else/finally의 각 블록과 assert(내부 가정)/raise(인터페이스 오류)를 구분하고, try 블록은 짧게 유지한다.
- with·contextlib로 자원 정리를 보장하고, Exception/BaseException 차이와 예외 변수 소멸 같은 함정에 주의한다.
- 제너레이터 자원은 밖에서 관리하고, __debug__ 비활성화와 eval/exec 사용은 피한다.
