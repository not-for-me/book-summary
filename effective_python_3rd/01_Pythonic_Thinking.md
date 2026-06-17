# 01. Pythonic Thinking

## 챕터 개요 (3줄 요약)
- 파이썬 커뮤니티가 경험을 통해 다듬어 온 "Pythonic" 스타일은 명시성, 단순함, 가독성을 최우선으로 한다.
- 이 챕터는 버전 확인, PEP 8, 표현식 단순화, 언패킹, 조건 표현식, 월러스 연산자, match 문 등 모든 코드에 영향을 주는 기본기를 다룬다.
- 다른 언어의 습관을 그대로 가져오지 말고, 가장 파이썬다운(the best) 방식으로 흔한 작업을 처리하는 법을 익히는 것이 핵심이다.

---

## Item 1: 사용 중인 Python 버전을 파악하라 (Know Which Version of Python You're Using)
> 시스템에 여러 Python 인터프리터가 공존할 수 있으므로, 실제로 어떤 버전이 실행되는지 명확히 확인해야 한다.

- 이 책의 예제는 Python 3.13(2024년 10월 릴리스) 기준이며 Python 2는 다루지 않는다.
- 명령행에서 `python`은 환경에 따라 `python2.7` 등 예상과 다른 버전을 가리킬 수 있다.
- `--version` 플래그나 `sys` 모듈로 런타임에 버전을 확인할 수 있다.
- Python 2는 2020년 4월(2.7.18)을 끝으로 공식 지원이 종료되었다.
- PyPy(Pretty Python) 같은 대체 런타임은 별도 명령(`pypy3`)을 사용한다.

```python
import sys

print(sys.platform)              # 예: darwin
print(sys.implementation.name)   # 예: cpython
major, minor, micro = sys.version_info[:3]
print(f"Running Python {major}.{minor}.{micro}")
```

## Item 2: PEP 8 스타일 가이드를 따르라 (Follow the PEP 8 Style Guide)
> 일관된 코드 스타일은 가독성과 협업을 크게 향상시킨다.

- PEP 8(Python Enhancement Proposal #8)은 코드 포매팅의 공식 스타일 가이드다.
- 들여쓰기는 스페이스 4칸, 한 줄은 79자 이하를 권장한다.
- 네이밍 규칙: 함수/변수는 `lowercase_underscore`, 클래스는 `CapitalizedWord`, 상수는 `ALL_CAPS`, 비공개 속성은 `__double_leading_underscore`.
- 빈 컨테이너 검사는 `len(x) == 0` 대신 `if not x`를 사용한다.
- `black`(자동 포매터)과 `pylint`(린터) 같은 도구로 PEP 8 준수를 자동화할 수 있다.

```python
# PEP 8 준수 예시
GREETING_PREFIX = "Hello"   # 모듈 상수: ALL_CAPS

def format_greeting(user_name: str) -> str:   # 함수: lowercase_underscore
    if not user_name:        # 빈 문자열 검사: len() 비교 대신 not 사용
        return GREETING_PREFIX
    return f"{GREETING_PREFIX}, {user_name}"
```

## Item 3: 컴파일 타임 에러 검출을 기대하지 마라 (Never Expect Python to Detect Errors at Compile Time)
> 파이썬은 동적 언어라서 거의 모든 에러를 런타임까지 미룬다.

- 로드 시 기본적인 SyntaxError(구문 오류)와 토큰화 오류만 검출된다.
- 할당되지 않은 변수, 0으로 나누기, 잘못된 인자 개수 등은 실행 전까지 잡히지 않는다.
- 동적 할당이 가능하기 때문에 정적 검사가 본질적으로 제한된다.
- `flake8` 린터나 `typing` 기반 타입 체커로 일부 에러를 사전에 잡을 수 있다.
- 따라서 런타임 가정 검증(assert)과 자동화된 테스트가 필수적이다.

```python
def maybe_define(flag):
    if flag:
        result = 42
    return result   # flag가 False면 런타임에 UnboundLocalError 발생

# 정적 분석 도구(mypy 등)로 사전 점검 권장
```

## Item 4: 복잡한 표현식 대신 헬퍼 함수를 작성하라 (Write Helper Functions Instead of Complex Expressions)
> 한 줄에 로직을 욱여넣기보다 헬퍼 함수로 분리하면 가독성과 재사용성이 좋아진다.

- `or` 연산자를 이용한 기본값 처리는 짧지만 읽기 어렵고 의도가 불명확하다.
- 같은 로직을 두세 번 이상 반복한다면 헬퍼 함수로 추출하는 것이 정답이다.
- DRY(Don't Repeat Yourself) 원칙을 따른다.
- 가독성에서 얻는 이득이 간결함의 이득을 항상 능가한다.
- 중간 변수로 표현식을 쪼개는 것도 좋은 전략이다.

```python
def get_first_int(values, key, default=0):
    found = values.get(key, [""])
    if found[0]:
        return int(found[0])
    return default

# 호출부가 훨씬 명확해진다
green = get_first_int(my_values, "green")
```

## Item 5: 인덱싱보다 다중 할당 언패킹을 선호하라 (Prefer Multiple-Assignment Unpacking over Indexing)
> 언패킹은 인덱싱보다 시각적 잡음이 적고 더 파이썬답다.

- 튜플은 불변 시퀀스이며 언패킹으로 여러 값을 한 번에 할당할 수 있다.
- 언패킹은 중첩된 이터러블(iterable)에도 일반화되어 적용된다.
- 임시 변수 없이 값을 교환(swap)할 수 있다: `a, b = b, a`.
- `enumerate`와 함께 for 루프 타깃에 언패킹을 쓰면 코드가 간결해진다.
- 인덱스 접근을 피하면 코드가 더 명료해진다.

```python
snacks = [("bacon", 350), ("donut", 240), ("muffin", 190)]

for rank, (name, calories) in enumerate(snacks, 1):
    print(f"#{rank}: {name} has {calories} calories")
```

## Item 6: 단일 원소 튜플은 항상 괄호로 감싸라 (Always Surround Single-Element Tuples with Parentheses)
> 후행 콤마 하나가 값을 의도치 않게 튜플로 바꿔버릴 수 있다.

- 단일 원소 튜플은 후행 콤마가 반드시 필요하다: `(1,)`.
- 괄호 없는 후행 콤마(`x = 1,`)는 찾기 어려운 버그를 만든다.
- 함수 호출 끝에 실수로 남은 콤마가 반환값을 튜플로 바꿀 수 있다.
- 자동 포매터나 정적 분석 도구가 이 문제를 드러내 줄 수 있다.
- 좌변/우변 어디서든 단일 원소 튜플은 괄호로 감싸는 것이 안전하다.

```python
to_refund = calculate_refund(value)   # 끝에 콤마가 있으면 tuple이 됨!
print(type(to_refund))                # 의도치 않게 <class 'tuple'>

correct_pair = (1,)   # 명확한 단일 원소 튜플
```

## Item 7: 간단한 인라인 로직에는 조건 표현식을 고려하라 (Consider Conditional Expressions for Simple Inline Logic)
> 조건 표현식은 if/else 동작을 표현식이 허용되는 거의 모든 곳에 넣을 수 있다.

- 형식: `참값 if 조건 else 거짓값` — C의 삼항 연산자와 순서가 다르다.
- 간단한 할당이나 함수 인자에는 간결하면서 명확하다.
- `and`/`or`를 이용한 불리언 트릭보다 훨씬 읽기 쉽고 오류가 적다.
- 여러 줄로 분리해야 할 만큼 길어지면 사용을 피해야 한다.
- 모호하거나 가독성을 해치면 일반 if 문이나 헬퍼 함수를 선택한다.

```python
i = 3
label = "even" if i % 2 == 0 else "odd"   # 참값이 먼저 온다 (C와 반대)

# 컴프리헨션 필터에서도 같은 순서
evens = [x / 4 for x in range(10) if x % 2 == 0]
```

## Item 8: 할당 표현식으로 반복을 제거하라 (Prevent Repetition with Assignment Expressions)
> 월러스 연산자(:=)는 할당과 평가를 한 번에 수행해 코드 중복을 줄인다.

- `:=`는 if/while 조건처럼 할당이 금지된 곳에서도 변수를 할당할 수 있게 한다.
- 변수의 영향 범위를 명확히 드러내 가독성을 높인다.
- 더 큰 표현식의 부분식일 때는 괄호로 감싸야 한다.
- switch/case나 do/while이 없는 파이썬에서 유사 동작을 깔끔히 구현한다.
- 여러 줄에서 같은 표현식을 반복할 때 사용을 고려한다.

```python
# do/while 흉내: 루프-앤-어-하프 관용구를 제거
bottles = []
while fresh_fruit := pick_fruit():
    for fruit, count in fresh_fruit.items():
        bottles.extend(make_juice(fruit, count))
```

## Item 9: 흐름 제어의 구조 분해에는 match를, if로 충분하면 피하라 (Consider match for Destructuring; Avoid When if Is Sufficient)
> match는 단순 분기보다는 복잡한 중첩 데이터의 구조 분해(destructuring)에 진가를 발휘한다.

- match(Python 3.10+)의 `case` 절은 단순 변수명을 캡처 패턴으로 취급해 오해를 부른다.
- 단순 분기를 if 대신 match로 바꾸면 오히려 오류가 발생하기 쉽다.
- 상수 비교에는 `enum`과 점(.) 연산자를 써야 캡처 패턴으로 오인되지 않는다.
- 딕셔너리, 리스트, 사용자 정의 클래스의 구조 분해에 강력하다.
- JSON(JavaScript Object Notation)처럼 반정형 데이터를 객체로 매핑할 때 특히 유용하다.

```python
def deserialize(record):
    match record:
        case {"customer": {"last": last, "first": first}}:
            return PersonCustomer(first, last)
        case {"customer": {"entity": company}}:
            return BusinessCustomer(company)
        case _:
            raise ValueError("Unknown record type")
```

---

## Summary (핵심 정리)
- Pythonic 코드는 명시성·단순함·가독성을 중시하며, 다른 언어의 습관 대신 파이썬다운 방식을 택한다.
- 복잡한 표현식은 헬퍼 함수로, 인덱싱은 언패킹으로, 반복 할당은 월러스 연산자로 정리해 가독성을 높인다.
- match는 단순 분기가 아닌 반정형/중첩 데이터의 구조 분해에 사용하고, 단순한 경우엔 if 문으로 충분하다.
