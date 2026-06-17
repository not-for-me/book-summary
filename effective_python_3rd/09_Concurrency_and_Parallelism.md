# 09. Concurrency and Parallelism

## 챕터 개요 (3줄 요약)
- 동시성(concurrency)은 여러 일을 동시에 하는 것처럼 보이게 하고, 병렬성(parallelism)은 실제로 동시에 실행해 속도를 높인다.
- 파이썬은 GIL(Global Interpreter Lock) 때문에 스레드로 진정한 병렬성을 얻기 어려우므로, 용도에 맞는 도구(스레드/큐/코루틴/프로세스)를 선택해야 한다.
- 블로킹 I/O엔 스레드, 대규모 동시 I/O엔 코루틴(asyncio), CPU 병렬엔 프로세스 풀을 사용한다.

---

## Item 67: 자식 프로세스 관리엔 subprocess를 사용하라 (Use subprocess to Manage Child Processes)
> subprocess 모듈은 자식 프로세스 실행·입출력 관리의 최선의 선택이다.

- 자식 프로세스는 파이썬 인터프리터와 병렬로 실행되어 CPU 코어를 활용한다.
- `subprocess.run`은 간단한 용도, `Popen`은 UNIX 파이프라인 같은 고급 용도에 쓴다.
- `poll`로 상태를 폴링하고 `communicate`로 종료를 기다린다.
- 여러 자식을 미리 시작하면 병렬로 실행된다.
- `communicate(timeout=...)`로 데드락·행을 방지한다.

```python
import subprocess
proc = subprocess.Popen(["sleep", "1"])
while proc.poll() is None:
    print("Working...")
proc.communicate(timeout=10)
```

## Item 68: 블로킹 I/O엔 스레드를, 병렬성엔 쓰지 마라 (Use Threads for Blocking I/O; Avoid for Parallelism)
> GIL 때문에 파이썬 스레드는 병렬 계산엔 무용하지만 블로킹 I/O엔 유용하다.

- CPython의 GIL은 한 번에 한 스레드만 진행하게 해 멀티코어 병렬을 막는다.
- 계산 집약 작업을 여러 스레드로 돌려도 속도가 빨라지지 않는다.
- 스레드는 여러 일을 동시에 하는 것처럼 보이게 하는 데 유용하다.
- 시스템 콜(블로킹 I/O) 직전 GIL을 풀어 I/O를 병렬화한다.
- 블로킹 I/O와 계산을 동시에 해야 할 때 스레드가 가장 간단하다.

```python
from threading import Thread
threads = [Thread(target=slow_systemcall) for _ in range(5)]
for t in threads: t.start()
for t in threads: t.join()   # I/O는 병렬로 진행
```

## Item 69: 스레드 데이터 경쟁을 막으려면 Lock을 사용하라 (Use Lock to Prevent Data Races in Threads)
> GIL이 있어도 데이터 경쟁은 발생하므로 뮤텍스(Lock)로 보호해야 한다.

- GIL은 데이터 구조를 보호하지 않으며, 스레드는 바이트코드 사이에서 중단될 수 있다.
- `counter += 1`은 읽기·계산·쓰기 3단계라 인터리빙 시 값이 손상된다.
- 선점(preemption) 때문에 원자적으로 보이는 연산도 중간에 멈출 수 있다.
- `threading.Lock`(뮤텍스)으로 한 번에 한 스레드만 접근하게 한다.
- `with lock:`으로 락 획득·해제를 명확히 한다.

```python
from threading import Lock
counter_lock = Lock()
def worker():
    global counter
    with counter_lock:
        counter += get_offset(data)
```

## Item 70: 스레드 간 작업 조율엔 Queue를 사용하라 (Use Queue to Coordinate Work Between Threads)
> queue.Queue는 블로킹·버퍼 크기·조인·셧다운 등 견고한 파이프라인 기능을 제공한다.

- 파이프라인은 직렬 단계들로 작업을 동시 처리하는 조립 라인이다.
- 직접 만든 큐는 바쁜 대기, 종료 신호, 메모리 폭발 문제가 있다.
- `Queue.get`은 데이터가 올 때까지 블로킹해 바쁜 대기를 없앤다.
- 최대 버퍼 크기로 백프레셔(back pressure)를 걸어 메모리 폭발을 막는다.
- `task_done`/`join`으로 진행을 추적하고 `shutdown`으로 워커를 종료한다.

```python
from queue import Queue, ShutDown
in_queue = Queue(100)   # 버퍼 크기로 백프레셔
def consumer():
    while True:
        try:
            item = in_queue.get()
        except ShutDown:
            return
        in_queue.task_done()
```

## Item 71: 동시성이 필요한 시점을 인식하라 (Know How to Recognize When Concurrency Is Necessary)
> 프로그램이 커지면 여러 동시 실행 흐름이 필요해지며, 팬아웃·팬인을 이해해야 한다.

- 단일 스레드에서 멀티 동시 실행으로 전환하는 것이 가장 어려운 변경이다.
- I/O가 직렬로 처리되면 셀 수에 비례해 지연이 선형 증가한다.
- 각 작업 단위마다 동시 실행 흐름을 만드는 것이 팬아웃(fan-out)이다.
- 모든 동시 작업의 완료를 기다리는 것이 팬인(fan-in)이다.
- 파이썬은 팬아웃·팬인을 위한 다양한 도구(스레드/큐/풀/코루틴)를 제공한다.

```python
# 콘웨이 생명게임: I/O가 필요해지면 동시성이 필요
async def game_logic(state, neighbors):
    data = await my_socket.read(50)   # 셀마다 I/O -> 병렬화 필요
    ...
```

## Item 72: 온디맨드 팬아웃에 새 Thread 인스턴스 생성을 피하라 (Avoid Creating New Thread Instances for On-Demand Fan-out)
> 작업마다 스레드를 만드는 것은 메모리·시작비용·디버깅 측면에서 나쁘다.

- 스레드는 Lock 등 조율 도구가 필요해 코드가 복잡해진다.
- 스레드당 약 8MB 메모리를 써서 수천 개면 메모리가 부족하다.
- 스레드 시작·컨텍스트 전환 비용이 지연을 늘린다.
- Thread는 예외를 호출자에게 되던지지 못해 디버깅이 어렵다.
- 동시 함수를 계속 만들고 끝낸다면 스레드는 해법이 아니다.

```python
# 안티패턴: 셀마다 스레드 생성 (수천 개면 메모리 폭발)
for y in range(height):
    for x in range(width):
        Thread(target=step_cell, args=(...)).start()
```

## Item 73: Queue 기반 동시성이 리팩터링을 요구함을 이해하라 (Understand How Using Queue for Concurrency Requires Refactoring)
> 고정 워커 + Queue는 확장성을 높이지만 상당한 리팩터링과 보일러플레이트를 요구한다.

- 고정 워커 스레드 + Queue는 자원 사용을 통제하고 시작 비용을 없앤다.
- 예외를 큐로 전파해 메인 스레드에서 재발생시켜 디버깅이 쉽다.
- 그러나 코드가 따라가기 어렵고 보조 기능이 필요하다.
- 병렬성 정도(워커 수)를 미리 지정해야 한다.
- 새 파이프라인 단계 추가 시 많은 변경과 보일러플레이트가 필요하다.

```python
def simulate_pipeline(grid, in_queue, out_queue):
    for y in range(grid.height):
        for x in range(grid.width):
            in_queue.put((y, x, state, neighbors))  # 팬아웃
    in_queue.join()
    # out_queue에서 팬인
```

## Item 74: 스레드가 필요한 동시성엔 ThreadPoolExecutor를 고려하라 (Consider ThreadPoolExecutor)
> ThreadPoolExecutor는 적은 리팩터링으로 I/O 병렬성과 예외 전파를 제공한다.

- `concurrent.futures.ThreadPoolExecutor`는 Thread와 Queue의 장점을 결합한다.
- 스레드를 미리 할당해 매번 시작 비용을 피한다.
- `max_workers`로 메모리 폭발을 방지한다.
- `Future.result()`가 예외를 자동으로 전파해 디버깅이 쉽다.
- 단, max_workers를 미리 지정해야 해 I/O 병렬성이 제한된다.

```python
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=10) as pool:
    futures = [pool.submit(step_cell, *args) for ...]  # 팬아웃
    for f in futures:
        f.result()   # 팬인 + 예외 전파
```

## Item 75: 코루틴으로 고도로 동시적인 I/O를 달성하라 (Achieve Highly Concurrent I/O with Coroutines)
> 코루틴은 메모리 오버헤드 없이 수만 개의 동시 함수를 실행해 I/O를 병렬화한다.

- 코루틴은 `async`/`await`로 정의하며 시작 비용이 함수 호출 수준이다.
- 활성 코루틴은 1KB 미만 메모리를 쓰고 await에서 일시 정지한다.
- 이벤트 루프가 적절히 작성된 함수들의 실행을 빠르게 인터리빙한다.
- `asyncio.gather`로 팬아웃·팬인하며 단일 스레드라 Lock이 불필요하다.
- 요구가 바뀌어도 async/await만 추가하면 되어 재구조화가 쉽다.

```python
import asyncio
async def simulate(grid):
    tasks = [step_cell(y, x, grid.get, next_grid.set)
             for y in range(grid.height) for x in range(grid.width)]
    await asyncio.gather(*tasks)   # 팬아웃 + 팬인
    return next_grid
```

## Item 76: 스레드 I/O를 asyncio로 포팅하는 법을 알라 (Know How to Port Threaded I/O to asyncio)
> 파이썬은 for/with/제너레이터/컴프리헨션의 비동기 버전을 제공해 포팅을 쉽게 한다.

- `async for`, `async with`, 비동기 컴프리헨션이 드롭인 대체로 제공된다.
- `__aiter__`, `aiter`, `anext`, `StopAsyncIteration` 등 비동기 대응물을 쓴다.
- `asyncio.start_server`, `open_connection`으로 소켓 보일러플레이트를 줄인다.
- `asynccontextmanager`로 비동기 컨텍스트 매니저를 만든다.
- `asyncio.create_task`로 이벤트 루프에 작업을 등록해 팬아웃한다.

```python
async def run_async_client(address):
    reader, writer = await asyncio.open_connection(*address)
    async with new_async_game(client, 1, 5, 3) as session:
        results = [outcome async for outcome in session]
```

## Item 77: 스레드와 코루틴을 혼합해 asyncio 전환을 쉽게 하라 (Mix Threads and Coroutines)
> run_in_executor와 run_coroutine_threadsafe로 점진적(top-down/bottom-up) 마이그레이션을 한다.

- 대규모 프로그램은 한 번에 포팅하기 어려워 점진적 전환이 필요하다.
- `loop.run_in_executor`로 코루틴이 동기 블로킹 함수를 스레드에서 실행한다(top-down).
- `run_coroutine_threadsafe`로 스레드가 코루틴을 이벤트 루프에서 실행한다.
- `loop.run_until_complete`로 동기 코드가 코루틴을 끝까지 실행한다(bottom-up).
- 두 접근은 결국 중간 지점에서 수렴한다.

```python
async def tail_async(handle, interval, write_func):
    loop = asyncio.get_event_loop()
    line = await loop.run_in_executor(None, readline, handle)
```

## Item 78: async 친화적 워커 스레드로 이벤트 루프 응답성을 극대화하라 (Maximize Responsiveness with async-Friendly Worker Threads)
> 코루틴 내 시스템 콜은 이벤트 루프를 막으므로 코루틴 친화 헬퍼 스레드 클래스를 쓴다.

- 코루틴 내 블로킹 I/O·스레드 시작은 응답성을 떨어뜨린다.
- `asyncio.run(..., debug=True)`로 이벤트 루프를 막는 코루틴을 탐지한다.
- 자체 이벤트 루프를 가진 Thread 서브클래스로 I/O를 캡슐화한다.
- `run_coroutine_threadsafe`+`wrap_future`로 Lock 없이 동기화한다.
- `__aenter__`/`__aexit__`로 async with에서 워커 스레드를 관리한다.

```python
async def run_fully_async(handles, interval, output_path):
    async with (WriteThread(output_path) as output,
                asyncio.TaskGroup() as group):
        for handle in handles:
            group.create_task(tail_async(handle, interval, output.write))
```

## Item 79: 진정한 병렬성엔 concurrent.futures를 고려하라 (Consider concurrent.futures for True Parallelism)
> ProcessPoolExecutor는 자식 인터프리터 프로세스로 GIL을 우회해 CPU 병렬을 달성한다.

- GIL은 스레드의 진정한 CPU 병렬을 막는다.
- `multiprocessing`은 자식 프로세스(별도 GIL)로 멀티코어를 활용한다.
- `ProcessPoolExecutor`로 한 줄만 바꿔 병렬 속도 향상을 얻는다.
- pickle로 직렬화·전송하므로 격리되고 고레버리지인 작업에 적합하다.
- 먼저 ThreadPoolExecutor → ProcessPoolExecutor 순으로 시도하고, multiprocessing 직접 사용은 마지막에 고려한다.

```python
from concurrent.futures import ProcessPoolExecutor
pool = ProcessPoolExecutor(max_workers=8)   # 이 한 줄로 병렬화
results = list(pool.map(gcd, NUMBERS))
```

---

## Summary (핵심 정리)
- GIL 때문에 스레드는 블로킹 I/O에만 유용하며, 데이터 경쟁은 Lock으로, 작업 조율은 Queue로 처리한다.
- 대규모 동시 I/O엔 코루틴(asyncio)이 메모리·시작비용 면에서 최적이며, 점진적 마이그레이션 도구가 잘 갖춰져 있다.
- 자식 프로세스(subprocess/ProcessPoolExecutor)로 GIL을 우회해 CPU 병렬성을 얻는다.
