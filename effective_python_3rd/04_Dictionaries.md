# 04. Dictionaries

## 챕터 개요 (3줄 요약)
- 딕셔너리(dictionary)는 키-값 매핑을 상수 시간(amortized)에 처리하는 핵심 자료구조로, 동적 상태 관리에 이상적이다.
- 누락된 키 처리(get, setdefault, defaultdict, __missing__)와 삽입 순서 보존 같은 특성을 정확히 이해해야 한다.
- 내부 상태가 복잡해지면 깊게 중첩된 딕셔너리 대신 클래스 계층으로 리팩터링하는 것이 유지보수에 유리하다.

---

## Item 25: 딕셔너리 삽입 순서에 의존할 때 주의하라 (Be Cautious when Relying on Dictionary Insertion Ordering)
> Python 3.7+부터 dict는 삽입 순서를 보존하지만, dict처럼 동작하는 사용자 타입은 그렇지 않을 수 있다.

- 3.5 이전엔 순서가 무작위였고, 3.7부터 삽입 순서 보존이 언어 명세가 되었다.
- 키워드 인자(**kwargs), 인스턴스 `__dict__`도 순서를 보존한다.
- 덕 타이핑(duck typing) 때문에 dict처럼 보이는 커스텀 컨테이너는 순서를 보장하지 않을 수 있다.
- 대응책 3가지: 순서에 의존하지 않게 작성, 런타임에 `isinstance(x, dict)` 검사, 타입 어노테이션+정적 분석.
- 빈번한 삽입/popitem이 필요하면 `collections.OrderedDict`가 성능상 유리할 수 있다.

```python
def get_winner(ranks):
    # 삽입 순서에 의존하지 않는 견고한 구현
    for name, rank in ranks.items():
        if rank == 1:
            return name
```

## Item 26: 누락 키 처리엔 in/KeyError보다 get을 선호하라 (Prefer get over in and KeyError)
> 단순 타입의 누락 키 처리에는 get 메서드가 가장 짧고 명확하다.

- 누락 키 처리 방법은 `in`, `KeyError`, `get`, `setdefault` 네 가지가 있다.
- 카운터처럼 단순 값에는 `get(key, 0)`이 한 번의 접근+할당으로 가장 간결하다.
- 리스트 값에는 get과 할당 표현식(:=)을 조합하면 가독성이 좋다.
- `setdefault`는 이름이 모호하고, 기본값을 매번 생성하는 비용 문제가 있다.
- `setdefault`가 적합해 보이면 대개 `defaultdict`가 더 낫다.

```python
counters = {"sourdough": 1}
count = counters.get("wheat", 0)   # 기본값 0
counters["wheat"] = count + 1

if (names := votes.get(key)) is None:   # 리스트 값 + 할당 표현식
    votes[key] = names = []
names.append(who)
```

## Item 27: 내부 상태의 누락 항목엔 setdefault보다 defaultdict를 선호하라 (Prefer defaultdict over setdefault)
> 딕셔너리 생성을 직접 통제할 때는 defaultdict가 불필요한 기본값 생성을 막아 효율적이다.

- `setdefault`는 키 존재 여부와 무관하게 매 호출마다 기본값 객체를 생성한다.
- `defaultdict`는 누락 키 접근 시 자동으로 기본값을 생성·저장한다.
- 생성자에 인자 없는 팩토리 함수(예: `set`, `list`)를 넘긴다.
- 내부 상태 관리에 defaultdict를 쓰면 코드가 짧고 효율적이다.
- 외부에서 받은 dict는 통제 불가하므로 get을 선호한다.

```python
from collections import defaultdict

class Visits:
    def __init__(self):
        self.data = defaultdict(set)
    def add(self, country, city):
        self.data[country].add(city)   # 키 자동 생성
```

## Item 28: __missing__으로 키 의존적 기본값을 구성하는 법을 알라 (Construct Key-Dependent Default Values with __missing__)
> 기본값 생성이 키에 의존하거나 비용이 크거나 예외를 던질 수 있을 때 __missing__을 쓴다.

- `setdefault`는 기본값 생성 비용이 크거나 예외 가능성이 있으면 부적합하다(open 항상 호출 등).
- `defaultdict`의 팩토리는 인자를 받지 못해 키에 의존하는 기본값을 만들 수 없다.
- dict를 상속해 `__missing__` 특수 메서드를 구현하면 키를 알고 기본값을 생성할 수 있다.
- `__missing__`은 값을 만들어 딕셔너리에 넣고 반환해야 한다.
- 이후 같은 키 접근에선 호출되지 않는다.

```python
class Pictures(dict):
    def __missing__(self, key):
        value = open(key, "a+b")   # 키(경로)에 의존하는 기본값
        self[key] = value
        return value
```

## Item 29: 깊게 중첩된 딕셔너리/리스트/튜플 대신 클래스를 조합하라 (Compose Classes Instead of Deeply Nesting)
> 내부 상태가 한 단계 이상 중첩되면 클래스 계층으로 분해하는 것이 가독성과 확장성에 좋다.

- 딕셔너리/튜플의 중첩이 깊어지면 코드가 취약하고 읽기 어려워진다.
- 중첩은 한 단계를 넘기지 않는 것이 좋다.
- 2-튜플보다 길어지면 다른 접근(클래스)을 고려한다.
- 불변 경량 데이터엔 `dataclasses`(`@dataclass(frozen=True)`)를 사용한다.
- 잘 정의된 인터페이스로 데이터를 캡슐화하고 추상화 계층을 만든다.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Grade:
    score: int
    weight: float

class Subject:
    def __init__(self):
        self._grades = []
    def report_grade(self, score, weight):
        self._grades.append(Grade(score, weight))
```

---

## Summary (핵심 정리)
- dict는 삽입 순서를 보존하지만 dict 유사 커스텀 타입엔 의존하지 말고, 누락 키는 get/defaultdict/__missing__을 상황에 맞게 쓴다.
- 단순 값엔 get, 내부 상태 관리엔 defaultdict, 키 의존 기본값엔 __missing__이 적합하다.
- 내부 상태가 복잡해지면 중첩 자료구조 대신 dataclass와 클래스 계층으로 리팩터링한다.
