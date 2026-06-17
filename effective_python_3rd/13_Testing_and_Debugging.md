# 13. Testing and Debugging

## 챕터 개요 (3줄 요약)
- 파이썬은 정적 타입 검사가 기본이 아니므로 코드 정확성 검증을 위한 테스트 작성이 특히 중요하다.
- unittest의 TestCase로 동작을 검증하고, 단위 테스트보다 통합 테스트를 선호하며, 필요시 Mock으로 복잡한 의존성을 시뮬레이션한다.
- 동적 특성을 활용해 pdb 대화형 디버깅과 tracemalloc 메모리 분석으로 문제를 효율적으로 추적한다.

---

## Item 108: TestCase 서브클래스에서 관련 동작을 검증하라 (Verify Related Behaviors in TestCase Subclasses)
> unittest의 TestCase를 상속하고 test로 시작하는 메서드로 동작별 테스트를 작성한다.

- 테스트는 `test`로 시작하는 메서드로 작성하며, 예외 없이 끝나면 통과다.
- `assertEqual`, `assertTrue` 등 헬퍼는 입력·출력을 출력해 `assert`보다 유용하다.
- `assertRaises`를 컨텍스트 매니저로 써서 예외 발생을 검증한다.
- test로 시작하지 않는 커스텀 헬퍼 메서드로 가독성을 높인다.
- `subTest`로 데이터 기반 테스트의 보일러플레이트를 줄인다.

```python
from unittest import TestCase, main

class UtilsTestCase(TestCase):
    def test_to_str_bytes(self):
        self.assertEqual("hello", to_str(b"hello"))
    def test_bad(self):
        with self.assertRaises(TypeError):
            to_str(object())
```

## Item 109: 단위 테스트보다 통합 테스트를 선호하라 (Prefer Integration Tests over Unit Tests)
> 파이썬의 동적 특성 때문에 컴포넌트 상호작용을 검증하는 통합 테스트가 가장 중요하다.

- 단위 테스트는 작은 부분을, 통합 테스트는 여러 컴포넌트의 협업을 검증한다.
- 파이썬은 정적 타입 보장이 없어 통합 테스트가 유일한 확신 수단일 때가 많다.
- 통합 테스트를 기본으로 하고, 엣지 케이스 많은 부분만 단위 테스트를 추가한다.
- Mock은 꼭 필요할 때만 쓰고 대부분 피하는 것이 좋다.
- 웹·DB 등과의 상호작용엔 더 큰 시스템 테스트가 필요하다.

```python
class ToasterIntegrationTest(TestCase):
    def test_wait_finish(self):
        self.toaster.push_down()
        self.assertTrue(self.toaster.hot)
        self.timer.timer.join()
        self.assertFalse(self.toaster.hot)  # mock 없이 end-to-end
```

## Item 110: setUp/tearDown/setUpModule/tearDownModule로 테스트를 격리하라 (Isolate Tests)
> setUp/tearDown으로 각 테스트를, 모듈 레벨 함수로 비싼 테스트 하네스를 관리한다.

- `setUp`/`tearDown`은 각 테스트 메서드 전후에 실행되어 테스트를 격리한다.
- 임시 디렉터리 생성·삭제 같은 환경 준비·정리에 쓴다.
- 비싼 통합 테스트 초기화는 매 테스트마다 하면 비현실적이다.
- `setUpModule`/`tearDownModule`은 모듈 전체에서 한 번만 실행된다.
- DB 시작 같은 무거운 하네스를 모듈 단위로 관리한다.

```python
def setUpModule():
    print("* Module setup")   # 모듈당 1회

class IntegrationTest(TestCase):
    def setUp(self):
        self.test_dir = TemporaryDirectory()   # 테스트마다
    def tearDown(self):
        self.test_dir.cleanup()
```

## Item 111: 복잡한 의존성이 있는 코드는 Mock으로 테스트하라 (Use Mocks to Test Code with Complex Dependencies)
> unittest.mock으로 DB 같은 의존성을 시뮬레이션해 테스트를 작성한다.

- `Mock(spec=...)`으로 함수·객체처럼 동작하는 모의 객체를 만든다.
- `return_value`로 반환값을, `side_effect`로 예외를 설정한다.
- `assert_called_once_with`로 호출 인자를 검증하고, `ANY`로 일부 인자를 무시한다.
- 모의 객체를 주입하는 방법: 키워드 전용 인자 또는 `unittest.mock.patch`.
- patch는 with·데코레이터·setUp에서 모듈/클래스 속성을 임시 교체한다.

```python
from unittest.mock import Mock
mock = Mock(spec=get_animals)
mock.return_value = expected
result = mock(database, "Meerkat")
mock.assert_called_once_with(database, "Meerkat")
```

## Item 112: 의존성을 캡슐화해 모킹과 테스트를 용이하게 하라 (Encapsulate Dependencies to Facilitate Mocking)
> 의존성을 클래스로 캡슐화하면 Mock 생성과 테스트가 쉬워진다.

- 단위 테스트에 반복 보일러플레이트가 많으면 의존성을 클래스로 캡슐화한다.
- `Mock(spec=ZooDatabase)`는 모든 메서드를 자동으로 모킹한다.
- spec은 오타난 메서드 호출을 잡아 코드·테스트 동시 버그를 막는다.
- end-to-end 테스트엔 의존성 주입을 위한 seam(헬퍼 함수)을 만든다.
- 테스트 가능하도록 코드를 리팩터링하는 것이 가치 있다.

```python
class ZooDatabase:
    def get_animals(self, species): ...
    def feed_animal(self, name, when): ...

database = Mock(spec=ZooDatabase)   # 모든 메서드 자동 모킹
```

## Item 113: 부동소수점 테스트엔 assertAlmostEqual로 정밀도를 제어하라 (Use assertAlmostEqual)
> assertAlmostEqual로 부동소수점 비교의 오차 허용 범위를 지정한다.

- 부동소수점은 연산 순서·반올림 때문에 미세하게 달라질 수 있다.
- `assertEqual`로 부동소수점을 비교하면 flaky 테스트가 된다.
- `assertAlmostEqual(a, b, places=2)`로 소수 자릿수 허용 범위를 지정한다.
- `delta=` 파라미터로 절대 오차 허용 범위를 지정한다(큰 수에 유용).
- `assertNotAlmostEqual`이나 `math.isclose`로 더 복잡한 경우를 다룬다.

```python
self.assertAlmostEqual(1.667, n / d, places=2)
self.assertAlmostEqual(90.9e6, a, delta=0.1e6)
```

## Item 114: pdb로 대화형 디버깅을 고려하라 (Consider Interactive Debugging with pdb)
> breakpoint()로 실행 중 프로그램 상태를 검사하고 단계별로 실행을 제어한다.

- `breakpoint()`(=`pdb.set_trace()`)를 코드에 넣어 디버거를 시작한다.
- `where`/`up`/`down`으로 콜 스택을 탐색한다.
- `step`/`next`/`return`/`continue`/`quit`로 실행을 제어한다.
- 조건부 `breakpoint()`로 특정 상황에서만 디버거를 띄운다.
- 사후 디버깅: `python -m pdb -c continue` 또는 `import pdb; pdb.pm()`.

```python
for got, wanted in zip(observed, ideal):
    err_2 = (got - wanted) ** 2
    if err_2 >= 1:
        breakpoint()   # 조건 충족 시 디버거 시작
```

## Item 115: tracemalloc로 메모리 사용·누수를 파악하라 (Use tracemalloc to Understand Memory Usage and Leaks)
> tracemalloc은 객체가 할당된 위치까지 추적해 메모리 사용·누수를 분석한다.

- CPython은 참조 카운팅+순환 탐지로 메모리를 관리하지만 누수가 생길 수 있다.
- `gc.get_objects()`는 객체 목록을 보여주나 할당 위치는 모른다.
- `tracemalloc`은 전후 스냅샷을 비교해 변화를 보여준다.
- `compare_to(..., "lineno")`로 메모리 사용 상위 코드를 찾는다.
- `"traceback"`으로 전체 스택 트레이스를 출력해 원인을 정확히 짚는다.

```python
import tracemalloc
tracemalloc.start(10)
time1 = tracemalloc.take_snapshot()
# ... 실행 ...
time2 = tracemalloc.take_snapshot()
for stat in time2.compare_to(time1, "lineno")[:3]:
    print(stat)
```

---

## Summary (핵심 정리)
- TestCase 서브클래스와 헬퍼 메서드로 동작을 검증하고, 단위 테스트보다 통합 테스트를 우선한다.
- setUp/tearDown으로 테스트를 격리하고, 복잡한 의존성은 Mock과 캡슐화로 다루며, 부동소수점은 assertAlmostEqual로 비교한다.
- 디버깅엔 pdb 대화형 디버거를, 메모리 분석엔 tracemalloc을 활용한다.
