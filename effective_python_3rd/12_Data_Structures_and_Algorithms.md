# 12. Data Structures and Algorithms

## 챕터 개요 (3줄 요약)
- 파이썬은 표준 자료구조·알고리즘의 최적화된 구현을 내장해 적은 노력으로 고성능을 달성하게 한다.
- 정렬(key/sort/sorted), 정렬된 시퀀스 검색(bisect), 큐(deque/heapq) 등을 알맞게 써서 알고리즘 복잡도를 개선한다.
- 날짜·시간(datetime), 정밀 계산(decimal), 직렬화(copyreg+pickle) 등 까다로운 작업도 내장 도구로 안전하게 처리한다.

---

## Item 100: key 파라미터로 복잡한 기준에 따라 정렬하라 (Sort by Complex Criteria Using the key Parameter)
> sort의 key 함수로 정렬에 사용할 값을 지정해 객체나 복합 기준 정렬을 구현한다.

- `sort`는 내장 타입은 자연 순서로 정렬하지만 객체는 비교 메서드가 없으면 실패한다.
- `key` 파라미터에 함수(보통 lambda)를 넘겨 정렬 기준 값을 반환한다.
- key에서 튜플을 반환하면 여러 기준을 우선순위대로 결합한다.
- 숫자형은 단항 음수(-)로 개별 기준의 방향을 뒤집을 수 있다.
- 음수가 불가능한 타입은 안정 정렬을 이용해 sort를 여러 번(낮은 우선순위→높은 우선순위) 호출한다.

```python
power_tools.sort(key=lambda x: (-x.weight, x.name))  # weight 내림, name 오름
```

## Item 101: sort와 sorted의 차이를 알라 (Know the Difference Between sort and sorted)
> sort는 제자리 정렬, sorted는 새 리스트를 반환하며 원본을 보존한다.

- `list.sort()`는 원본을 제자리에서 수정해 메모리·속도 효율이 좋다.
- `sorted()`는 정렬된 새 리스트를 반환하고 원본은 그대로 둔다.
- sorted는 모든 이터러블(튜플·딕트·셋)에 동작한다.
- 둘 다 `reverse`, `key` 파라미터를 지원한다.
- 원본 보존·유연성이 필요하면 sorted, 성능·메모리가 중요하면 sort.

```python
alphabetical = sorted(original)   # 원본 보존
original.sort()                   # 제자리 정렬
```

## Item 102: 정렬된 시퀀스 검색엔 bisect를 고려하라 (Consider Searching Sorted Sequences with bisect)
> bisect_left는 정렬된 리스트를 로그 시간에 이진 검색해 선형 검색보다 훨씬 빠르다.

- `list.index`나 선형 스캔은 길이에 비례하는 선형 시간이 걸린다.
- `bisect.bisect_left`는 정렬된 시퀀스를 이진 검색한다.
- 정확히 일치하거나 삽입 위치(가장 가까운 값)를 반환한다.
- 복잡도가 로그라 100만 개 검색이 선형 20개 검색과 비슷한 시간이다.
- list뿐 아니라 시퀀스처럼 동작하는 임의 객체에도 쓸 수 있다.

```python
from bisect import bisect_left
index = bisect_left(data, 91234.56)   # 가장 가까운 위치 (로그 시간)
```

## Item 103: 생산자-소비자 큐엔 deque를 선호하라 (Prefer deque for Producer-Consumer Queues)
> list.pop(0)은 길이에 따라 초선형으로 느려지므로 FIFO 큐엔 deque가 이상적이다.

- FIFO(선입선출) 큐는 생산자가 append, 소비자가 처리하는 구조다.
- list의 `pop(0)`은 모든 항목을 앞으로 이동시켜 초선형(quadratic)으로 느려진다.
- `collections.deque`는 양방향 큐로 양끝 삽입·삭제가 상수 시간이다.
- 소비자는 `popleft()`를 호출한다.
- 성능이 중요한 FIFO 큐엔 deque가 최적이다.

```python
import collections
queue = collections.deque()
queue.append(item)      # 생산자: 상수 시간
email = queue.popleft() # 소비자: 상수 시간
```

## Item 104: 우선순위 큐엔 heapq를 사용하는 법을 알라 (Know How to Use heapq for Priority Queues)
> heapq는 중요도 순으로 항목을 처리하는 우선순위 큐를 효율적으로(로그 시간) 구현한다.

- 우선순위 큐는 FIFO가 아니라 상대적 중요도 순으로 처리한다.
- list+sort로 구현하면 큐가 커질수록 초선형으로 느려진다.
- `heapq`의 `heappush`/`heappop`은 추가·최소 항목 제거가 로그 복잡도다.
- 항목은 자연 순서가 필요하므로 `@functools.total_ordering`+`__lt__`를 정의한다.
- `heapify`로 리스트를 선형 시간에 힙으로 만든다.

```python
from heapq import heappush, heappop
import functools

@functools.total_ordering
class Book:
    def __lt__(self, other):
        return self.due_date < other.due_date

heappush(queue, book)
overdue = heappop(queue)   # 가장 이른 마감일 (로그 시간)
```

## Item 105: 로컬 시계엔 time 대신 datetime을 사용하라 (Use datetime Instead of time for Local Clocks)
> datetime+zoneinfo는 플랫폼 독립적으로 시간대 변환을 안정적으로 처리한다.

- `time` 모듈은 플랫폼 의존적이라 시간대 변환이 불안정하다.
- `datetime` 클래스와 `zoneinfo` 모듈로 시간대 변환을 신뢰성 있게 한다.
- 항상 UTC로 표현하고 표시 직전에만 로컬 시간으로 변환한다.
- `ZoneInfo("US/Eastern")` 같은 시간대 DB를 사용한다.
- Windows 등에선 `tzdata` 패키지가 필요할 수 있다.

```python
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

nyc_dt = naive.replace(tzinfo=ZoneInfo("US/Eastern"))
utc_dt = nyc_dt.astimezone(timezone.utc)        # UTC로 변환
sf_dt = utc_dt.astimezone(ZoneInfo("US/Pacific"))
```

## Item 106: 정밀도가 중요하면 decimal을 사용하라 (Use decimal when Precision Is Paramount)
> Decimal은 고정소수점 연산과 반올림 제어로 IEEE 754 부동소수점 오차를 해결한다.

- 부동소수점은 IEEE 754 표현 때문에 미세한 오차가 생긴다(예: 5.365 → 5.364...).
- `decimal.Decimal`은 기본 28자리 고정소수점 연산을 제공한다.
- 정확성이 중요하면 항상 `str`로 생성한다(`Decimal("1.45")`).
- `quantize`로 원하는 자릿수와 반올림 방식(ROUND_UP 등)을 제어한다.
- 무한 정밀 유리수는 `fractions.Fraction`을 고려한다.

```python
from decimal import Decimal, ROUND_UP
cost = Decimal("1.45") * Decimal(222) / Decimal(60)
rounded = cost.quantize(Decimal("0.01"), rounding=ROUND_UP)
```

## Item 107: copyreg로 pickle 직렬화를 유지보수 가능하게 하라 (Make pickle Serialization Maintainable with copyreg)
> copyreg로 직렬화·역직렬화 함수를 등록해 클래스 변경에 대한 하위 호환성을 보장한다.

- `pickle`은 신뢰하는 프로그램 간 객체 전송에만 사용한다(보안상 안전하지 않음).
- 클래스 속성이 추가·제거되면 이전 pickle 객체의 역직렬화가 깨진다.
- `copyreg.pickle`로 직렬화 함수를 등록하고 기본값 생성자로 누락 속성을 채운다.
- version 파라미터로 필드 제거 같은 비호환 변경을 마이그레이션한다.
- 안정적인 import 경로로 클래스 이름 변경에 대응한다.

```python
import copyreg

def pickle_game_state(state):
    kwargs = state.__dict__
    kwargs["version"] = 2
    return unpickle_game_state, (kwargs,)

copyreg.pickle(GameState, pickle_game_state)
```

---

## Summary (핵심 정리)
- 정렬은 key 파라미터와 sort/sorted 구분을 활용하고, 정렬된 데이터 검색엔 bisect로 로그 시간을 달성한다.
- FIFO 큐엔 deque, 우선순위 큐엔 heapq를 써서 초선형 성능 저하를 피한다.
- 시간은 datetime+zoneinfo로, 정밀 계산은 decimal로, pickle 직렬화 유지보수는 copyreg로 안전하게 처리한다.
