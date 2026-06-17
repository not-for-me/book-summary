# 02. Strings and Slicing

## 챕터 개요 (3줄 요약)
- 파이썬은 텍스트 처리에 강력하며, bytes(8비트 이진 데이터)와 str(유니코드 코드 포인트)를 명확히 구분해 다뤄야 한다.
- 문자열 포매팅은 f-string이 가장 표현력 있고 간결하며, C 스타일(%)과 str.format은 피하는 것이 좋다.
- 시퀀스의 슬라이싱과 언패킹을 올바르게 사용하면 인덱스 오류를 줄이고 가독성을 크게 높일 수 있다.

---

## Item 10: bytes와 str의 차이를 알라 (Know the Differences Between bytes and str)
> bytes는 raw 8비트 값을, str은 유니코드 코드 포인트를 담으며 둘은 호환되지 않는다.

- str은 인코딩이 없고 bytes는 텍스트 인코딩이 없다. 변환은 `encode`/`decode`로 한다.
- 유니코드 샌드위치(Unicode sandwich): 입출력 경계에서만 인코딩/디코딩하고 코어는 str을 사용한다.
- bytes와 str은 `+`, `>`, `==`, `%` 등 연산자에서 함께 쓸 수 없다(같은 ASCII여도 == 는 False).
- 파일 입출력 시 이진 데이터는 반드시 "rb"/"wb" 모드로 연다.
- 시스템 기본 인코딩(보통 UTF-8)에 의존하지 말고 `open`에 `encoding`을 명시한다.

```python
def to_str(value):
    if isinstance(value, bytes):
        return value.decode("utf-8")
    return value

def to_bytes(value):
    if isinstance(value, str):
        return value.encode("utf-8")
    return value

with open("data.bin", "wb") as f:   # 이진 모드 명시
    f.write(b"\xf1\xf2\xf3")
```

## Item 11: C 스타일/str.format보다 f-string을 선호하라 (Prefer Interpolated F-Strings)
> f-string은 표현력, 간결함, 명료함을 모두 갖춘 최고의 포매팅 방식이다.

- C 스타일(%)은 타입/순서 불일치 오류, 가독성 저하, 값 반복, 장황함의 4가지 문제가 있다.
- `str.format`은 미니 언어를 도입했지만 핵심 문제를 그대로 남겨 권장하지 않는다.
- f-string은 `f"..."` 접두사로 시작하며 현재 스코프의 이름을 직접 참조한다.
- 플레이스홀더 안에 임의의 파이썬 표현식을 넣어 즉석 변형이 가능하다.
- 포맷 스펙(콜론 뒤)에도 표현식을 넣을 수 있다(예: 자릿수 파라미터화).

```python
key, value = "my_var", 1.234
print(f"{key:<10} = {value:.2f}")     # my_var     = 1.23

places, number = 3, 1.23456
print(f"My number is {number:.{places}f}")   # 1.235
```

## Item 12: 출력 시 repr과 str의 차이를 이해하라 (Understand the Difference Between repr and str)
> str은 사람이 읽기 좋은 형태를, repr은 디버깅용 명확한 표현(타입 구분 포함)을 제공한다.

- `print`/`str`은 타입 정보를 숨겨 `5`(int)와 `"5"`(str)를 구분하지 못한다.
- 디버깅 시에는 `repr()` 또는 f-string의 `!r` 변환을 써서 타입 차이를 드러낸다.
- 내장 타입의 repr 결과는 종종 `eval`로 원본 객체를 복원할 수 있는 유효한 표현식이다.
- 사용자 클래스의 기본 repr은 무의미하므로 `__repr__`을 직접 정의한다.
- 사람이 읽을 별도 형식이 필요하면 `__str__`을 추가로 정의한다.

```python
class BetterClass:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __repr__(self):
        return f"BetterClass({self.x!r}, {self.y!r})"

print(repr(BetterClass(2, "bar")))   # BetterClass(2, 'bar')
```

## Item 13: 리스트 안에서는 특히 명시적 문자열 연결을 선호하라 (Prefer Explicit String Concatenation)
> 인접한 문자열 리터럴은 자동으로 이어지지만, 리스트/튜플에서는 의도가 모호해진다.

- `"a" "b"`는 `"a" + "b"`와 같은 암시적 연결(implicit concatenation)이다.
- 리스트 안에서 콤마를 실수로 빼면 두 줄이 조용히 합쳐지는 버그가 생긴다.
- 콤마를 실수로 추가하면 의도와 달리 튜플이 만들어진다.
- 리스트/튜플 리터럴에서는 항상 명시적 `+` 연산자를 사용한다.
- 함수 호출에서 위치 인자가 여러 개면 명시적 연결을 쓴다(단일 위치 인자는 암시적도 무방).

```python
lines = [
    "first line\n",
    "second line\n" +     # 명시적 연결로 의도 분명
    "third line\n",
]
```

## Item 14: 시퀀스 슬라이싱 방법을 알라 (Know How to Slice Sequences)
> 슬라이싱은 `somelist[start:end]`로 시퀀스의 일부를 손쉽게 다룬다.

- start는 포함, end는 제외이며 시작은 0, 끝은 len을 생략해 잡음을 줄인다.
- 음수 인덱스로 끝에서부터의 오프셋을 표현할 수 있다(`a[-3:]`).
- 범위를 벗어난 인덱스는 슬라이싱에서 조용히 무시되어 최대 길이 처리가 쉽다.
- 슬라이싱 결과는 새 리스트이며 원본은 변하지 않는다.
- 슬라이스에 할당하면 길이가 달라도 해당 범위를 교체한다.

```python
a = ["a", "b", "c", "d", "e", "f", "g", "h"]
print(a[2:5])      # ['c', 'd', 'e']
print(a[-3:])      # ['f', 'g', 'h']
a[2:5] = [99, 22]  # 길이 달라도 교체 가능
```

## Item 15: 한 표현식에서 스트라이딩과 슬라이싱을 함께 쓰지 마라 (Avoid Striding and Slicing in a Single Expression)
> `somelist[start:end:stride]`에 세 값을 모두 넣으면 매우 읽기 어렵다.

- stride는 n번째마다 항목을 선택한다(`x[::2]`는 짝수 위치).
- `x[::-1]`은 역순이지만 UTF-8로 인코딩된 bytes에서는 깨진다.
- 음수 stride와 start/end가 결합되면 동작을 예측하기 어렵다.
- 가능하면 양수 stride만 단독으로 쓰고 start/end는 생략한다.
- 셋 다 필요하면 두 번의 할당으로 나누거나 `itertools.islice`를 쓴다.

```python
x = ["a", "b", "c", "d", "e", "f", "g", "h"]
strided = x[::2]    # ['a', 'c', 'e', 'g']
sliced = strided[1:-1]   # ['c', 'e'] — 두 단계로 분리해 명확하게
```

## Item 16: 슬라이싱보다 캐치올 언패킹을 선호하라 (Prefer Catch-All Unpacking over Slicing)
> 별표 표현식(*)을 쓰면 길이를 몰라도 나머지를 한 변수로 모을 수 있다.

- 기본 언패킹은 시퀀스 길이를 미리 알아야 하지만 별표는 그 제약을 없앤다.
- 별표 표현식은 시작/중간/끝 어느 위치에도 올 수 있다.
- 인덱스/슬라이스 기반 분할보다 off-by-one 오류가 적다.
- 별표 부분은 항상 list가 되며, 남은 항목이 없으면 빈 리스트다.
- 이터레이터에도 쓸 수 있으나, 메모리에 다 담길 때만 사용해야 한다.

```python
car_ages_desc = [20, 19, 15, 9, 8, 7, 6, 4, 1, 0]
oldest, second, *others = car_ages_desc
# 헤더와 나머지 행 분리
header, *rows = iter(generate_csv())
```

---

## Summary (핵심 정리)
- 텍스트는 bytes(이진)와 str(유니코드)를 명확히 구분하고, 인코딩/디코딩은 입출력 경계에서만 수행한다.
- 문자열 포매팅은 f-string을 기본으로 삼고, 디버깅에는 repr/!r을 활용한다.
- 슬라이싱은 간결하게, 스트라이딩과 결합은 피하며, 길이 불명 분할에는 별표 캐치올 언패킹을 사용한다.
