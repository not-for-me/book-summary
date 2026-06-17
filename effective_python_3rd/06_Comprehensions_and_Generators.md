# 06. Comprehensions and Generators

## 챕터 개요 (3줄 요약)
- 컴프리헨션(comprehension)은 리스트·딕트·셋을 간결하게 생성하는 문법으로 가독성을 크게 높인다.
- 제너레이터(generator)는 yield로 값을 점진적으로 반환해 메모리 효율과 가독성을 개선한다.
- 제너레이터 합성(yield from), 양방향 통신(send/throw)의 함정을 이해하고 더 단순한 대안을 택해야 한다.

---

## Item 40: map/filter 대신 컴프리헨션을 사용하라 (Use Comprehensions Instead of map and filter)
> 컴프리헨션은 lambda가 필요 없어 map/filter보다 명료하다.

- 리스트 컴프리헨션은 `[식 for x in seq]` 형태로 한 줄에 파생 리스트를 만든다.
- `if` 절로 입력 항목을 손쉽게 필터링할 수 있다(map은 filter 없이는 불가).
- 딕셔너리·셋 컴프리헨션도 동일하게 지원된다.
- map/filter는 생성자로 감싸야 해 가독성이 떨어진다.
- 단, 컴프리헨션은 전체 결과를 메모리에 구체화하므로 대용량엔 주의한다.

```python
a = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
even_squares = [x**2 for x in a if x % 2 == 0]
squares_dict = {x: x**2 for x in a if x % 2 == 0}
```

## Item 41: 컴프리헨션에서 제어 부분식을 2개 이하로 제한하라 (Avoid More Than Two Control Subexpressions)
> 컴프리헨션에 루프·조건이 3개 이상이면 읽기 어려워진다.

- 컴프리헨션은 여러 단계의 루프와 레벨별 다중 조건을 지원한다.
- 같은 레벨의 다중 `if`는 암묵적 `and`로 결합된다.
- 2단계 평탄화나 구조 복제 정도는 합리적이다.
- 제어 부분식(루프+조건)이 2개를 넘으면 피해야 한다.
- 더 복잡하면 일반 for/if 문과 헬퍼 함수를 쓴다.

```python
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = [x for row in matrix for x in row]        # 2개: OK
squared = [[x**2 for x in row] for row in matrix] # OK
# 3개 이상의 중첩은 일반 루프로 작성 권장
```

## Item 42: 할당 표현식으로 컴프리헨션의 반복을 줄여라 (Reduce Repetition in Comprehensions with Assignment Expressions)
> 월러스 연산자(:=)로 같은 계산을 한 번만 수행해 가독성과 성능을 개선한다.

- 컴프리헨션에서 동일 계산이 조건과 값에 반복되면 잡음·버그 위험이 생긴다.
- 할당 표현식으로 값을 한 번 계산해 변수에 저장하고 재사용한다.
- 변수를 조건에서 정의해야 값 표현식에서 안전하게 참조된다.
- 월러스로 정의한 변수는 둘러싼 스코프로 누출된다(루프 변수는 누출 안 됨).
- 제너레이터 표현식에서도 동일하게 동작한다.

```python
found = {name: batches for name in order
         if (batches := get_batches(stock.get(name, 0), 8))}
```

## Item 43: 리스트 반환 대신 제너레이터를 고려하라 (Consider Generators Instead of Returning Lists)
> yield로 결과를 점진 생성하면 코드가 명료하고 메모리가 제한된다.

- 결과를 list에 append해 반환하면 코드가 빽빽하고 잡음이 많다.
- 제너레이터는 yield 표현식으로 값을 하나씩 생성한다.
- 호출 시 즉시 실행되지 않고 이터레이터를 반환한다.
- list 결과 누적이 없어 임의 크기 입력도 처리 가능하다.
- 반환된 이터레이터는 상태가 있어 재사용할 수 없다.

```python
def index_words_iter(text):
    if text:
        yield 0
    for index, letter in enumerate(text):
        if letter == " ":
            yield index + 1
```

## Item 44: 큰 리스트 컴프리헨션엔 제너레이터 표현식을 고려하라 (Consider Generator Expressions for Large List Comprehensions)
> 제너레이터 표현식은 결과를 구체화하지 않아 대용량 입력에 메모리 효율적이다.

- 리스트 컴프리헨션은 대용량 입력에서 메모리를 과도하게 쓴다.
- `()`로 감싸면 제너레이터 표현식이 되어 이터레이터를 반환한다.
- 한 번에 한 항목씩 생성하므로 메모리 오버헤드가 작다.
- 제너레이터 표현식끼리 합성(체이닝)할 수 있다.
- 체이닝은 빠르고 메모리 효율적이나 이터레이터는 1회성이다.

```python
it = (len(x) for x in open("my_file.txt"))
roots = ((x, x**0.5) for x in it)   # 합성
print(next(roots))
```

## Item 45: yield from으로 여러 제너레이터를 합성하라 (Compose Multiple Generators with yield from)
> yield from은 중첩 제너레이터를 단일 제너레이터로 합성하는 보일러플레이트를 제거한다.

- 여러 제너레이터를 for+yield로 직접 잇는 것은 반복적이고 잡음이 많다.
- `yield from`은 중첩 제너레이터의 모든 값을 yield한 뒤 제어를 돌려준다.
- 수동 중첩 루프와 yield 보일러플레이트를 없앤다.
- 결과는 동일하면서 약간 더 빠르게 실행된다.
- 제너레이터를 합성할 때 적극 사용하길 권한다.

```python
def animate_composed():
    yield from move(4, 5.0)
    yield from pause(3)
    yield from move(2, 3.0)
```

## Item 46: send 메서드 대신 이터레이터를 제너레이터 인자로 넘겨라 (Pass Iterators into Generators Instead of send)
> send와 yield from을 함께 쓰면 예기치 않은 None이 나오므로 입력 이터레이터 전달이 낫다.

- `send`는 yield 표현식에 값을 주입해 양방향 채널을 만든다.
- 첫 send는 반드시 None이어야 한다(아직 yield 미도달).
- send와 yield from을 결합하면 자식 전환마다 None이 출력되는 함정이 있다.
- send는 우변 yield 사용이 비직관적이라 가독성이 나쁘다.
- 대신 입력 이터레이터를 제너레이터에 넘겨 cascade로 처리하는 것이 명료하다.

```python
def wave_cascading(amplitude_it, steps):
    step_size = 2 * math.pi / steps
    for step in range(steps):
        amplitude = next(amplitude_it)   # 입력 이터레이터에서 받음
        yield amplitude * math.sin(step * step_size)
```

## Item 47: throw 메서드 대신 클래스로 반복 상태 전이를 관리하라 (Manage State Transitions with a Class Instead of throw)
> throw는 가독성을 해치므로 상태를 가진 클래스로 상태 전이를 구현하는 것이 좋다.

- `throw`는 가장 최근 yield 위치에서 예외를 재발생시킨다.
- 제너레이터는 try/except로 주입된 예외를 잡을 수 있다.
- 양방향 통신에 유용하지만 중첩과 보일러플레이트가 늘어 읽기 어렵다.
- 상태와 전이 메서드를 가진 클래스로 대체하면 훨씬 명료하다.
- `__bool__` 등 특수 메서드로 자연스러운 인터페이스를 만든다.

```python
class Timer:
    def __init__(self, period):
        self.current = self.period = period
    def reset(self):
        self.current = self.period
    def tick(self):
        before = self.current
        self.current -= 1
        return before
    def __bool__(self):
        return self.current > 0
```

---

## Summary (핵심 정리)
- 파생 자료구조는 map/filter 대신 컴프리헨션으로, 제어 부분식은 2개 이하로 제한하고 반복 계산은 월러스로 줄인다.
- 대용량·스트림 처리는 리스트 대신 제너레이터/제너레이터 표현식을 쓰고, 합성엔 yield from을 활용한다.
- send와 throw는 함정이 많으므로 입력 이터레이터 전달과 상태 클래스 같은 단순한 대안을 택한다.
