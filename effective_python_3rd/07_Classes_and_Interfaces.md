# 07. Classes and Interfaces

## 챕터 개요 (3줄 요약)
- 파이썬은 상속·다형성·캡슐화를 지원하는 객체지향 언어이면서, 일급 함수를 통한 함수형 스타일도 권장한다.
- 간단한 인터페이스는 함수로, 데이터 중심 클래스는 dataclasses로 작성해 보일러플레이트를 줄인다.
- super를 통한 부모 초기화, 믹스인 합성, 공개 속성 선호, collections.abc 상속 등으로 견고하고 확장 가능한 클래스를 만든다.

---

## Item 48: 간단한 인터페이스엔 클래스 대신 함수를 받아라 (Accept Functions Instead of Classes for Simple Interfaces)
> 파이썬의 일급 함수 덕분에 단순한 훅(hook)은 함수로 구현하는 것이 명료하다.

- 많은 내장 API가 동작 커스터마이즈를 위해 함수(훅)를 받는다(예: `sort(key=...)`).
- 함수와 메서드 참조는 일급이라 변수처럼 전달할 수 있다.
- 상태가 필요하면 stateful 클로저나 클래스를 쓸 수 있다.
- `__call__` 특수 메서드로 인스턴스를 함수처럼 호출 가능하게 한다.
- 상태 유지가 필요하면 클로저보다 `__call__` 클래스가 의도를 더 명확히 드러낸다.

```python
class BetterCountMissing:
    def __init__(self):
        self.added = 0
    def __call__(self):
        self.added += 1
        return 0

counter = BetterCountMissing()
result = defaultdict(counter, current)   # __call__에 의존
```

## Item 49: isinstance 검사 함수보다 객체지향 다형성을 선호하라 (Prefer OO Polymorphism over isinstance Checks)
> 거대한 if/isinstance 분기 대신 다형성으로 런타임에 적절한 구현을 디스패치한다.

- `isinstance`로 타입별 분기를 할 수 있지만 거대한 함수가 되기 쉽다.
- 다형성은 메서드 호출을 가장 구체적인 서브클래스 구현으로 디스패치한다.
- 각 타입이 자기 데이터 옆에 동작(메서드)을 캡슐화한다.
- 모든 결정이 런타임에 이뤄져 임의 중첩을 추가 복잡도 없이 지원한다.
- 다형성 코드는 isinstance보다 읽기·유지·확장·테스트가 쉽다.

```python
class Node:
    def evaluate(self):
        raise NotImplementedError

class AddNode(Node):
    def __init__(self, left, right):
        self.left, self.right = left, right
    def evaluate(self):
        return self.left.evaluate() + self.right.evaluate()
```

## Item 50: OO 다형성 대신 functools.singledispatch를 고려하라 (Consider functools.singledispatch)
> 동작이 독립적이고 데이터를 공유할 때는 단일 디스패치로 관련 동작을 한곳에 모은다.

- OOP는 클래스 중심 조직화라 동작이 여러 모듈에 흩어진다.
- 단일 디스패치(single dispatch)는 첫 인자 타입으로 함수 버전을 선택한다.
- `functools.singledispatch` 데코레이터와 `.register`로 타입별 구현을 등록한다.
- 클래스를 수정하지 않고 동작을 추가할 수 있다.
- 새 타입 추가 시 각 디스패치 함수 구현을 추가해야 하는 트레이드오프가 있다.

```python
import functools

@functools.singledispatch
def my_evaluate(node):
    raise NotImplementedError

@my_evaluate.register(Add)
def _(node):
    return my_evaluate(node.left) + my_evaluate(node.right)
```

## Item 51: 경량 클래스 정의엔 dataclasses를 선호하라 (Prefer dataclasses for Defining Lightweight Classes)
> dataclass 데코레이터는 __init__, __repr__, __eq__ 등 반복 코드를 자동 생성한다.

- 속성을 타입 힌트와 함께 한 번만 선언해 오타·순서 오류를 줄인다.
- `kw_only=True`로 키워드 전용 초기화를, 기본값과 `field(default_factory=...)`로 가변 기본값을 처리한다.
- `__repr__`, `__eq__`를 자동 제공한다.
- `asdict`, `astuple` 헬퍼로 직렬화·변환이 쉽다.
- `order=True`로 비교·정렬을 자동 지원한다.

```python
from dataclasses import dataclass, field

@dataclass(kw_only=True)
class DataclassRGBA:
    red: int
    green: int
    blue: int
    alpha: float = 1.0
    tags: list = field(default_factory=list)
```

## Item 52: @classmethod 다형성으로 객체를 일반적으로 생성하라 (Use @classmethod Polymorphism to Construct Objects Generically)
> 파이썬은 생성자가 하나뿐이므로, 대체 생성자는 @classmethod로 다형적으로 만든다.

- 파이썬은 클래스당 `__init__` 하나만 지원한다.
- `@classmethod`로 대체 생성자(factory)를 정의한다.
- 클래스 메서드 다형성으로 여러 구체 서브클래스를 일반적으로 생성·연결한다.
- `cls`를 통해 서브클래스에 맞는 인스턴스를 만든다.
- 글루 코드를 다시 쓰지 않고 새 서브클래스를 추가할 수 있다.

```python
class GenericInputData:
    @classmethod
    def generate_inputs(cls, config):
        raise NotImplementedError

class PathInputData(GenericInputData):
    @classmethod
    def generate_inputs(cls, config):
        for name in os.listdir(config["data_dir"]):
            yield cls(os.path.join(config["data_dir"], name))
```

## Item 53: super로 부모 클래스를 초기화하라 (Initialize Parent Classes with super)
> super와 MRO(메서드 결정 순서)는 초기화 순서와 다이아몬드 상속 문제를 해결한다.

- 부모 `__init__`을 직접 호출하면 다중 상속에서 순서·중복 호출 문제가 생긴다.
- 다이아몬드 상속 시 공통 조상이 여러 번 초기화될 수 있다.
- `super()`는 C3 선형화 기반 MRO로 공통 조상을 한 번만 실행한다.
- 인자 없는 `super().__init__()`을 사용하는 것이 유지보수에 좋다.
- `__mro__`로 초기화 순서를 확인할 수 있다.

```python
class TimesSevenCorrect(MyBaseClass):
    def __init__(self, value):
        super().__init__(value)
        self.value *= 7
```

## Item 54: 믹스인 클래스로 기능을 합성하는 것을 고려하라 (Consider Composing Functionality with Mix-in Classes)
> 믹스인(mix-in)은 인스턴스 속성 없이 소수의 메서드만 제공해 다중 상속의 골칫거리를 피한다.

- 믹스인은 자체 인스턴스 속성이나 `__init__`을 요구하지 않는다.
- 동적 검사(hasattr, isinstance, __dict__)로 제네릭 기능을 한 번만 작성한다.
- 플러그형 동작으로 서브클래스별 커스터마이즈가 가능하다.
- 인스턴스 메서드와 클래스 메서드를 모두 추가할 수 있다.
- 믹스인을 합성·계층화해 간단한 동작에서 복잡한 기능을 만든다.

```python
class ToDictMixin:
    def to_dict(self):
        return self._traverse_dict(self.__dict__)

class JsonMixin:
    def to_json(self):
        return json.dumps(self.to_dict())
```

## Item 55: 비공개 속성보다 공개 속성을 선호하라 (Prefer Public Attributes over Private Ones)
> 비공개 속성은 강제되지 않으며, 서브클래스 확장을 어렵게 만들 뿐이다.

- 비공개(`__name`)는 `_ClassName__name`으로 이름 변환될 뿐 강제되지 않는다.
- "We are all consenting adults here" — 확장을 막지 않는 것이 파이썬 철학이다.
- 내부 API는 단일 밑줄(`_protected`) 규칙으로 표시한다.
- 비공개 속성을 쓰면 서브클래스 오버라이드가 취약해진다.
- 서브클래스 이름 충돌 방지가 필요할 때만 비공개를 고려한다.

```python
class ApiClass:
    def __init__(self):
        self.__value = 5   # 이름 충돌 방지용 비공개
    def get(self):
        return self.__value
```

## Item 56: 불변 객체 생성엔 dataclasses를 선호하라 (Prefer dataclasses for Creating Immutable Objects)
> frozen=True 데이터클래스로 불변 객체를 만들면 함수형 스타일과 견고함을 얻는다.

- 불변 객체는 생성 후 변경을 막아 디버깅하기 어려운 버그를 줄인다.
- `@dataclass(frozen=True)`로 수정 시 `FrozenInstanceError`를 발생시킨다.
- 정적 분석 도구가 수정 시도를 사전에 탐지한다.
- `dataclasses.replace`로 일부 속성만 바꾼 복사본을 만든다.
- frozen 데이터클래스는 `__eq__`, `__hash__`를 자동 제공해 딕트 키·셋 멤버로 쓸 수 있다.

```python
from dataclasses import dataclass, replace

@dataclass(frozen=True)
class Point:
    name: str
    x: float
    y: float

moved = replace(p, x=p.x + 10)   # 변경된 복사본
```

## Item 57: 커스텀 컨테이너 타입은 collections.abc를 상속하라 (Inherit from collections.abc Classes)
> 컨테이너 타입을 직접 만들 땐 collections.abc 추상 기반 클래스를 상속해 누락 메서드를 보완한다.

- 단순 용도엔 `list`, `dict`를 직접 상속해 기본 동작을 활용한다.
- 컨테이너를 직접 구현하면 `__getitem__`, `__len__` 등 많은 특수 메서드가 필요하다.
- `__getitem__`만으론 `len`, `count`, `index` 등이 제공되지 않는다.
- `collections.abc`의 `Sequence`, `MutableMapping` 등을 상속하면 누락 메서드를 알려준다.
- 필수 메서드만 구현하면 나머지 메서드를 자동으로 얻는다.

```python
from collections.abc import Sequence

class BetterNode(SequenceNode, Sequence):
    pass   # __getitem__, __len__만 구현하면 index, count 자동 제공
```

---

## Summary (핵심 정리)
- 간단한 인터페이스엔 함수/__call__을, 타입 분기엔 다형성이나 singledispatch를, 데이터 클래스엔 dataclasses를 사용한다.
- 부모 초기화는 super로, 기능 합성은 믹스인으로, 속성은 공개를 선호하고 불변 객체는 frozen 데이터클래스로 만든다.
- 커스텀 컨테이너는 collections.abc를 상속해 표준 동작을 안전하게 확보한다.
