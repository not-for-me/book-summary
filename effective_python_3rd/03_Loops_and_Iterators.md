# 03. Loops and Iterators

## 챕터 개요 (3줄 요약)
- 파이썬은 명령형 루프와 함수형 스타일의 이터레이터(iterator)를 모두 지원하여 순차 데이터 처리를 유연하게 한다.
- enumerate, zip, any/all 같은 내장 함수로 루프를 더 간결하고 효율적으로(단락 평가 포함) 작성할 수 있다.
- 이터레이터의 1회성 소비, 반복 중 컨테이너 수정 같은 함정을 이해하고 방어적으로 코딩해야 한다.

---

## Item 17: range보다 enumerate를 선호하라 (Prefer enumerate over range)
> 인덱스와 값을 동시에 얻을 때는 range+인덱싱 대신 enumerate가 명료하다.

- range는 정수 시퀀스 반복에 유용하지만, 리스트 인덱싱과 결합하면 장황하다.
- enumerate는 이터레이터를 게으른(lazy) 제너레이터로 감싸 (인덱스, 값) 쌍을 yield한다.
- for 문에서 쌍을 바로 언패킹할 수 있어 코드가 깔끔해진다.
- 두 번째 인자로 시작 카운트를 지정할 수 있다(기본 0).
- 인덱스가 필요한 거의 모든 경우 enumerate가 더 읽기 쉽다.

```python
flavors = ["vanilla", "chocolate", "pecan"]
for rank, flavor in enumerate(flavors, 1):
    print(f"{rank}: {flavor}")
```

## Item 18: 병렬 처리에는 zip을 사용하라 (Use zip to Process Iterators in Parallel)
> 여러 관련 리스트를 동시에 순회할 때 zip이 인덱싱의 시각적 잡음을 제거한다.

- zip은 둘 이상의 이터레이터를 감싸 각 항목의 튜플을 yield하는 게으른 제너레이터다.
- 인덱스 접근 없이 for 문에서 직접 언패킹할 수 있다.
- 한 항목씩 소비하므로 무한 길이 입력에도 안전하다.
- 길이가 다르면 가장 짧은 이터레이터에 맞춰 조용히 잘린다.
- `strict=True`(3.10+)로 길이 불일치 시 예외를 발생시킬 수 있고, `zip_longest`로 기본값을 채울 수도 있다.

```python
names = ["Cecilia", "Lise", "Marie"]
counts = [len(n) for n in names]
for name, count in zip(names, counts, strict=True):
    print(f"{name}: {count}")
```

## Item 19: for/while 루프 뒤의 else 블록을 피하라 (Avoid else Blocks After for and while Loops)
> 루프 뒤 else는 break가 없을 때 실행되어 직관과 반대로 동작한다.

- 루프의 else 블록은 루프가 break 없이 완료되면 실행된다.
- break가 실행되면 else는 건너뛴다.
- 빈 시퀀스나 처음부터 False인 while에서도 else가 즉시 실행된다.
- 검색 로직에 쓸 수 있지만 동작이 비직관적이라 혼란을 준다.
- 헬퍼 함수(조기 return 또는 플래그 변수)로 대체하는 것이 명확하다.

```python
def coprime(a, b):
    for i in range(2, min(a, b) + 1):
        if a % i == 0 and b % i == 0:
            return False
    return True   # else 블록 대신 조기 return 패턴
```

## Item 20: 루프 종료 후 루프 변수를 사용하지 마라 (Never Use for Loop Variables After the Loop Ends)
> 루프 변수는 종료 후에도 남지만, 루프가 한 번도 실행되지 않으면 정의되지 않는다.

- for 루프 변수는 루프 종료 후 현재 스코프에 그대로 남는다.
- 그러나 이터러블이 비어 한 번도 반복하지 않으면 변수가 없어 NameError가 난다.
- 따라서 루프 후 변수 존재를 항상 보장할 수 없어 위험하다.
- 컴프리헨션과 제너레이터 표현식은 루프 변수를 누출하지 않는다.
- 예외 핸들러 변수도 누출되지 않는다(별도 주의 필요).

```python
# 안티패턴: 빈 리스트면 i가 정의되지 않음
def find_index(items, target):
    for i, name in enumerate(items):
        if name == target:
            return i
    return len(items)   # 루프 변수 의존 대신 명시적 반환
```

## Item 21: 인자를 순회할 때 방어적으로 하라 (Be Defensive when Iterating over Arguments)
> 이터레이터는 단 한 번만 소비되므로 여러 번 순회하는 함수는 문제가 생긴다.

- 제너레이터를 인자로 받아 두 번 순회하면 두 번째엔 빈 결과가 나온다.
- 소진된 이터레이터를 순회해도 예외가 아닌 빈 결과를 반환해 혼란스럽다.
- `list()`로 복사하면 되지만 대용량에서 메모리 위험이 있다.
- `__iter__`를 제너레이터로 구현한 컨테이너 클래스를 만들면 매번 새 이터레이터를 제공한다.
- `iter(x) is x` 또는 `isinstance(x, Iterator)`로 이터레이터를 거부할 수 있다.

```python
class ReadVisits:
    def __init__(self, path):
        self.path = path
    def __iter__(self):              # 매 호출마다 새 이터레이터
        with open(self.path) as f:
            for line in f:
                yield int(line)
```

## Item 22: 순회 중 컨테이너를 수정하지 마라; 복사본이나 캐시를 써라 (Never Modify Containers While Iterating)
> 순회 도중 컨테이너 크기를 바꾸면 예측하기 어려운 런타임 오류가 발생한다.

- dict/set에 순회 중 키를 추가/삭제하면 RuntimeError가 발생한다(값만 변경은 허용).
- list에 현재 위치 앞에 삽입하면 무한 루프, 뒤에 append는 허용되는 등 비일관적이다.
- 가장 안전한 규칙: 순회 중에는 컨테이너를 절대 수정하지 않는다.
- 수정이 필요하면 복사본을 순회하고 원본을 수정한다.
- 성능이 중요하면 별도 컨테이너에 변경을 모았다가 `update`로 병합한다.

```python
my_dict = {"red": 1, "blue": 2, "green": 3}
for key in list(my_dict.keys()):   # 키 복사본을 순회
    if key == "blue":
        my_dict["green"] = 4       # 원본 수정은 안전
```

## Item 23: 효율적 단락 평가를 위해 any와 all에 이터레이터를 넘겨라 (Pass Iterators to any and all)
> any/all에 제너레이터를 넘기면 조건 충족 즉시 멈추는 단락 평가가 가능하다.

- all은 모든 항목이 truthy면 True, falsey를 만나면 즉시 False로 멈춘다.
- any는 반대로 truthy를 만나면 즉시 True로 멈춘다.
- and/or과 달리 항상 불리언 True/False를 반환한다.
- 리스트 컴프리헨션 대신 제너레이터 표현식을 넘겨야 효율 이점을 얻는다.
- 조기에 True가 필요하면 any, 조기에 False가 필요하면 all을 쓴다(드모르간 법칙으로 상호 변환).

```python
# 제너레이터 표현식 -> 첫 False에서 즉시 멈춤
all_heads = all(flip_is_heads() for _ in range(20))
# 잘못된 예: 리스트 컴프리헨션은 20번 모두 평가 후 전달
# all_heads = all([flip_is_heads() for _ in range(20)])
```

## Item 24: 이터레이터/제너레이터 작업에 itertools를 고려하라 (Consider itertools)
> itertools는 이터레이터 연결·필터링·조합을 위한 강력한 함수를 제공한다.

- 연결: `chain`(이어붙이기), `repeat`, `cycle`, `tee`(병렬 분할), `zip_longest`.
- 필터링: `islice`(복사 없는 슬라이싱), `takewhile`, `dropwhile`, `filterfalse`.
- 조합: `batched`(고정 크기 묶음), `pairwise`(인접 쌍), `accumulate`(누적), `product`(곱집합).
- `permutations`, `combinations`, `combinations_with_replacement`로 순열·조합 생성.
- 까다로운 반복 코드를 만나면 공식 문서를 다시 확인할 가치가 있다.

```python
import itertools
print(list(itertools.chain([1, 2], [3, 4])))        # [1, 2, 3, 4]
print(list(itertools.batched([1, 2, 3, 4, 5], 2)))  # [(1,2),(3,4),(5,)]
print(list(itertools.pairwise(["A", "B", "C"])))    # [('A','B'),('B','C')]
```

---

## Summary (핵심 정리)
- enumerate와 zip으로 인덱싱을 줄이고, 루프 뒤 else나 루프 변수 재사용 같은 비직관적 패턴은 피한다.
- 이터레이터는 1회성이므로 여러 번 순회 시 방어적으로 컨테이너를 받고, 순회 중 수정은 복사본/캐시로 처리한다.
- any/all로 효율적 단락 평가를, itertools로 복잡한 이터레이터 연산을 간결하게 구현한다.
