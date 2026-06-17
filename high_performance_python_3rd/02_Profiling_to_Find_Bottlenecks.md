# 02. Profiling to Find Bottlenecks

## 챕터 개요 (3줄 요약)

- 프로파일링(profiling)은 최소한의 노력으로 가장 큰 성능 이득을 얻기 위해 병목(bottleneck)을 찾는 작업이며, 직관에 의존하지 말고 가설을 세운 뒤 측정 결과로 검증해야 한다.
- 거친 측정(timeit, time, cProfile)에서 세밀한 측정(line_profiler, memory_profiler, Scalene)으로 단계적으로 좁혀가며, 실행 중 프로세스는 py-spy, 시간축 호출 스택은 VizTracer로 분석한다.
- CPython 바이트코드(dis 모듈)를 이해하면 어떤 코딩 스타일이 왜 느린지 알 수 있고, 최적화 중에는 반드시 단위 테스트로 정확성을 유지해야 한다.

---

## 1. Profiling Efficiently & The Julia Set (효율적 프로파일링과 줄리아 집합)

> 대표성 있는(representative) 시스템에서 느린 부분을 찾아야 하며, 프로파일링은 10~100배의 오버헤드를 유발하므로 테스트 대상을 격리(isolate)해야 한다.

- 줄리아 집합(Julia Set)은 CPU 바운드이면서 RAM도 소비하는 프랙탈 예제로, 각 픽셀을 독립 계산하는 "embarrassingly parallel" 문제다.
- 각 좌표는 `z = z*z + c`를 반복하며 `abs(z) < 2` 탈출 조건(escape condition)을 maxiter까지 검사한다.
- 흰색 영역(많은 반복)은 계산이 비싸고, 검은색 영역(적은 반복)은 빠르므로 비선형(nonlinear) 동작을 보인다.
- 이 예제는 의도적으로 비효율적으로 작성되어 느린 로직 문장과 메모리 소비 문장을 식별할 수 있게 한다.
- 결정론적(deterministic) 코드이므로 출력 합계로 알고리즘이 깨지지 않았는지 sanity check 할 수 있다.

---

## 2. Simple Timing — print, Decorator, Unix time (간단한 타이밍 측정)

> 가장 간단한 측정은 print, 데코레이터(decorator), 그리고 Unix의 `/usr/bin/time` 명령이며, 실행 시간은 항상 변동하므로 정상 변동폭을 관찰해야 한다.

- print 문은 빠르고 직관적이지만 stdout을 어지럽히므로 일회성 조사에만 적합하다.
- 데코레이터는 함수 위에 한 줄만 추가해 자동으로 시간을 측정하며, `@wraps(fn)`로 원본 함수명과 docstring을 보존한다.
- `timeit` 모듈은 가비지 컬렉터(garbage collector)를 일시 비활성화하므로 실제 동작과 다를 수 있고, 최선(best) 결과만 취한다.
- `timeit.py`는 최솟값을, IPython의 `%timeit`은 평균±표준편차를 쓰므로 두 방식을 섞어 비교하면 안 된다.
- Unix `time -p`는 real(벽시계), user(CPU 사용자 시간), sys(커널 시간)를 보여주며 Python 외 시작 시간도 포함한다.
- `--verbose` 옵션의 Maximum resident set size는 최대 RAM 사용량을, Major page faults는 디스크 스왑 여부를 알려준다.

### 타이밍 데코레이터 예제 (원문 의도 유지, 새 예제로 재작성)

```python
import time
from functools import wraps

def timer(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()       # time.time()보다 정밀
        result = fn(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"@timer: {fn.__name__} took {elapsed:0.4f} seconds")
        return result
    return wrapper

@timer
def heavy_sum(limit: int) -> int:
    return sum(i * i for i in range(limit))
```

---

## 3. cProfile & line_profiler (함수/라인 단위 프로파일링)

> cProfile은 함수 단위로 어떤 함수가 가장 오래 걸리는지 고수준 그림을 주고, line_profiler는 선택한 함수를 라인 단위로 세밀하게 분석한다.

- cProfile은 CPython 가상머신에 후킹되어 모든 함수 호출 시간을 측정하며 `-s cumulative`로 누적 시간 정렬이 가능하다.
- `-o profile.stats`로 통계 파일을 저장한 뒤 `pstats`로 분석하고, `print_callers()`/`print_callees()`로 호출 관계를 추적한다.
- SnakeViz는 cProfile 출력을 박스 크기로 시각화해 새 프로젝트의 CPU 동작을 직관적으로 파악하게 해준다.
- line_profiler는 `@profile` 데코레이터와 `kernprof -l -v`로 실행하며 각 라인의 호출 횟수와 시간 비율(% Time)을 보여준다.
- 줄리아 예제에서 while 테스트가 44% 시간을 차지했고, `n += 1`조차 Python의 동적 룩업(`__add__` 탐색) 때문에 비쌌다.
- 가장 싼 조건(`n < maxiter`)을 and의 왼쪽에 두면 단락 평가(short-circuit)로 미세한 속도 이득을 얻을 수 있다(반드시 측정 필요).

### line_profiler 사용 예제

```python
# pip install line_profiler  →  kernprof -l -v script.py
@profile                      # kernprof가 주입하는 데코레이터
def count_escapes(coords, c, maxiter):
    counts = [0] * len(coords)
    for idx, z in enumerate(coords):
        n = 0
        while n < maxiter and abs(z) < 2:   # 싼 조건을 왼쪽에
            z = z * z + c
            n += 1
        counts[idx] = n
    return counts
```

---

## 4. Memory Profiling — memory_profiler, Scalene (메모리 프로파일링)

> memory_profiler는 RAM 사용량을 라인 단위로 측정하고, Scalene은 CPU·메모리·GPU 프로파일링을 가벼운 오버헤드로 통합한다.

- 메모리 측정은 CPU만큼 명확하지 않은데, 메모리 과할당(over-allocation)과 비순간적 가비지 컬렉션 때문에 한 라인이 결정론적 양을 할당하지 않는다.
- 따라서 한 라인보다 여러 라인의 총 추세(gross trend)를 보는 것이 더 유용하며, Mem usage 컬럼을 신뢰해야 한다.
- `mprof run` + `mprof plot`은 시간축으로 샘플링해 거의 오버헤드 없이 RAM 증가를 시각화한다.
- zs·cs 리스트를 미리 만들지 않고 좌표를 즉석 계산하면 RAM을 140MB → 60MB로 절반 줄일 수 있다.
- Scalene은 자체 경량 프로파일러로 20% 미만 오버헤드를 가지며, Python 시간·네이티브 시간·시스템 시간을 구분한다.
- Scalene/Copilot 같은 GenAI 제안은 오래되고 잘못될 수 있으니 항상 회의적으로 직접 벤치마크해야 한다.

### 메모리 절약 패턴 (즉석 생성으로 리스트 보관 회피)

```python
# 나쁨: 큰 리스트 두 개를 미리 만들어 RAM 점유
zs = [complex(x, y) for y in ys for x in xs]

# 좋음: 즉석 생성으로 RAM 절약 (제너레이터/루프 내부 생성)
def iter_grid(xs, ys):
    for y in ys:
        for x in xs:
            yield complex(x, y)   # 메모리에 전부 쌓지 않음
```

---

## 5. py-spy, VizTracer, Bytecode (실행 중 프로세스 · 시간축 · 바이트코드)

> py-spy는 코드 수정 없이 실행 중인 프로세스를 들여다보고, VizTracer는 시간축 호출 스택을 보여주며, dis 모듈은 CPython 바이트코드를 드러낸다.

- py-spy는 Rust로 작성된 샘플링 프로파일러로 거의 런타임 영향이 없으며, PID로 운영 환경의 장기 실행 프로세스를 분석할 수 있다.
- py-spy는 `top` 형태 표시와 flame chart(불꽃 차트) 내보내기를 지원한다.
- VizTracer는 시간을 왼쪽→오른쪽으로 표시하며 함수 호출·인자·반환값을 추적하지만, 정보량이 많으면 "Circular buffer is full" 오류가 난다.
- dis 모듈은 스택 기반 CPython 가상머신의 바이트코드를 보여주어 왜 어떤 스타일이 느린지 이해하게 해준다.
- Python 3.11+는 "hot" 코드를 특수화(specialization)하며, Specialist 도구는 특수화 성공(녹색)/실패(빨강) 영역을 색으로 표시한다.
- 일반적으로 바이트코드 라인이 많을수록 느리므로, 내장 함수(`sum(range(n))`)가 직접 작성한 루프보다 2배 이상 빠르다.

### 표현적 vs 간결한 코드의 바이트코드 차이

```python
# 더 많은 바이트코드 → 느림
def sum_loop(upper=1_000_000):
    total = 0
    for n in range(upper):
        total += n
    return total

# 최적화된 C 구현 호출 → 빠름
def sum_terse(upper=1_000_000):
    return sum(range(upper))
```

---

## 6. Unit Testing & Profiling Strategy (단위 테스트와 프로파일링 전략)

> 최적화 중에는 반드시 단위 테스트로 정확성을 유지하고, 안정적 벤치마크를 위해 CPU 가속 기능과 배경 작업을 제어해야 한다.

- 테스트 없이 최적화하면 알고리즘을 깨뜨린 것을 속도 향상으로 착각하기 쉬우므로 단위 테스트와 coverage.py가 필수다.
- `@profile` 데코레이터는 테스트 시 NameError를 내므로, 도구가 주입하지 않았을 때 동작하는 no-op 데코레이터를 추가한다.
- 안정적 벤치마크를 위해 Turbo Boost/SpeedStep 비활성화, AC 전원 사용, 백업·Dropbox 종료, 다회 반복 측정을 권장한다.
- 테스트 대상 코드를 본체에서 분리하면 다른 스레드·프로세스·네트워크·디스크의 side effect를 피할 수 있다.
- 수치 최적화에서는 부동소수점 결과를 텍스트 파일로 출력해 `diff`로 미세한 반올림 오차를 즉시 잡는다.

### no-op @profile 데코레이터 패턴

```python
# line_profiler/memory_profiler 미사용 시 더미 데코레이터 주입
if 'line_profiler' not in dir() and 'profile' not in dir():
    def profile(func):
        def inner(*args, **kwargs):
            return func(*args, **kwargs)
        return inner
```

---

## Summary (핵심 정리)

- 프로파일링의 핵심은 직관 대신 가설→측정→검증의 흐름이며, 거친 도구(timeit/time/cProfile)로 병목을 좁힌 뒤 세밀한 도구(line_profiler/memory_profiler/Scalene)로 라인 단위 원인을 찾는 것이다.
- CPU와 RAM은 측정 특성이 다르므로(메모리는 추세로 봐야 함), py-spy(실행 중)·VizTracer(시간축)·dis(바이트코드)를 상황에 맞게 조합해 "under the hood"를 이해해야 한다.
- 모든 최적화는 단위 테스트로 정확성을 보장하고 안정적 벤치마크 환경(Turbo Boost 끄기, 다회 반복)에서 측정해야 신뢰할 수 있으며, 한 번에 한 가지 가설만 검증하는 것이 원칙이다.
