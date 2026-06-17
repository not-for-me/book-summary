# 09. Asynchronous I/O

## 챕터 개요 (3줄 요약)

- I/O 바운드 프로그램은 디스크·네트워크 같은 느린 장치에서 데이터를 기다리는 "I/O wait"에 시간을 낭비하며, 비동기 I/O(asynchronous I/O)는 이 대기 시간에 다른 작업을 실행해 활용한다.
- 동시성(concurrency)은 단일 스레드·단일 CPU·공유 메모리에서 작업들의 죽은 시간을 인터리빙하는 것으로, 각자 자원을 갖는 병렬성(parallelism)과 다르며 이벤트 루프(event loop)로 관리된다.
- Python의 async/await와 asyncio·aiohttp로 동시 요청을 쌓으면 직렬 대비 큰 속도 향상을 얻고, CPU+I/O 혼합 워크로드에선 배칭(batching)이나 완전 비동기로 I/O를 CPU 작업 뒤에 숨길 수 있다.

---

## 1. Concurrency & Event Loop (동시성과 이벤트 루프)

> I/O 작업마다 커널에 요청하고 결과를 기다리는데, 네트워크 쓰기 1ms 동안 2.4GHz CPU는 240만 명령어를 실행할 수 있을 만큼 I/O는 CPU보다 느리다.

- I/O wait 동안 실행이 일시정지되며, 직렬 실행은 이 페널티를 매번 치른다.
- 동시성은 같은 스레드에서 작업들의 대기 시간을 인터리빙해 공유 자원을 나눠 쓰는 것으로, 멀티프로세싱/멀티스레딩과 달리 새 프로세스를 만들지 않는다.
- 이벤트 루프는 실행할 함수들의 리스트로, 비차단(nonblocking) 비동기 호출은 즉시 반환하고 나중에 완료 이벤트가 발생한다.
- 함수 전환에는 비용이 있으므로, 동시성은 I/O wait가 많을 때 가장 효과적이다.
- 동시 코드는 메모리를 공유해 멀티스레드보다 쓰기 쉽지만, 어떤 라인이 언제 실행될지 모르므로 경쟁 상태(race condition)에 주의해야 한다.

### 이벤트 루프 개념도

```
   event loop queue: [ task_A, task_B, task_C, ... ]
        |
   run task_A --> hits I/O wait --> yields control back
        |
   run task_B (while A's I/O completes in background)
        |
   ... I/O done event fires --> resume A
```

---

## 2. How async/await Works (async/await의 동작 원리)

> async def로 정의한 코루틴(coroutine)은 제너레이터와 같은 방식으로 구현되어, await는 yield처럼 실행을 일시정지하고 결과가 준비되면 재개한다.

- 콜백(callback) 패러다임은 함수가 값을 반환하는 대신 콜백을 호출하며, 긴 체인이 "callback hell"이 되어 흐름 추적이 어렵다.
- Python 3.4의 asyncio와 PEP 492로 futures 메커니즘과 await/async 키워드가 도입되어 직렬 코드처럼 읽히는 비동기 코드가 가능해졌다.
- 비동기 함수는 Future 객체(미래 결과의 약속)를 반환하며, await로 채워질 때까지 기다리되 그 동안 다른 계산을 할 수 있다.
- Future는 결과나 코드 호출을 담지 않으며, 이벤트 루프가 스케줄링할 때까지 실제 코드는 실행되지 않는다.
- functools.partial로 인자를 고정해 인자 없는 task를 만드는 패턴이 비동기에서 자주 쓰인다.

### async/await (원문 의도 유지, 새 예제)

```python
import asyncio

async def fetch_and_store(value):
    payload = f"Hello {value}"
    # await가 실행을 일시정지하고 이벤트 루프에 제어를 넘김
    response = await save_to_db(payload)
    return response

# 임시 이벤트 루프로 코루틴 실행
asyncio.run(fetch_and_store("World"))
```

---

## 3. Serial vs Async Web Crawler (직렬 vs 비동기 크롤러)

> 직렬 크롤러는 요청을 하나씩 순차 처리해 1000개 요청에 약 102초가 걸리지만, aiohttp 비동기 버전은 동시에 처리해 약 1.33초(약 76.6배)로 줄인다.

- aiohttp는 aio-libs의 일부로 asyncio를 친숙하게 노출하며, HTTP/2나 sync/async 공용이 필요하면 httpx도 선택지다.
- asyncio.run으로 sync↔async를 전환하지만, 비동기 함수 내 코드는 async 실행 중에만 쓸 수 있다.
- coro 객체 생성 시 process 함수는 실행되지 않으며, TaskGroup이 컨텍스트 종료 시 await할 때 비로소 실행된다.
- 코루틴은 await할 때만 이벤트 루프에 제어를 넘기므로, 루프 안에 await가 없으면 task가 실행되지 않는다.
- aiohttp.ClientSession은 기본 100개 동시 요청 제한이 있어, 100개씩 묶여 100ms 지연을 한꺼번에 느낀다.
- 동시 요청 수를 늘리면 약 250개 이후 수익이 체감하며, CPU 바운드·컨텍스트 스위칭 오버헤드가 한계가 된다(기본 100~200 권장).

### 비동기 크롤러 (원문 의도 유지, 새 예제)

```python
import asyncio, aiohttp

async def fetch_size(session, url):
    async with session.get(url) as resp:
        return len(await resp.text())

async def crawl(urls):
    async with aiohttp.ClientSession() as session:
        async with asyncio.TaskGroup() as tg:   # Python 3.11+
            tasks = [tg.create_task(fetch_size(session, u)) for u in urls]
    return sum(t.result() for t in tasks)

asyncio.run(crawl(url_list))
```

---

## 4. Shared CPU–I/O Workload (CPU+I/O 혼합 워크로드)

> bcrypt 해시 계산(CPU) + DB 저장(I/O) 혼합 문제에서, 직렬은 I/O wait에 시간을 낭비하지만 배칭과 완전 비동기로 I/O를 숨길 수 있다.

- 직렬 버전은 난이도 8로 600회 반복에 70.6초가 걸리는데, 그중 약 85%가 I/O wait다.
- CPU 작업이 길어질수록 직렬 I/O의 상대적 손해는 줄어들므로, 최적화 전에 워크로드 특성을 이해해야 한다.
- 배칭(pipelining)은 결과를 모아 작은 비동기 버스트로 보내며, 100개씩 묶으면 난이도 8에서 10.21초(약 6.95배)가 된다.
- 배칭 효과는 DB 처리량에 좌우되며, DB가 한 번에 1개만 처리하면 직렬과 다를 바 없다.
- 완전 비동기는 CPU 작업 중에 I/O를 동시 수행해 I/O를 총 런타임에서 거의 숨기며, 직렬보다 약 7.3배, 배칭보다 I/O가 약 2배 빠르다.
- CPU 바운드 코드엔 await가 없으므로 `await asyncio.sleep(0)`로 50~100ms마다 이벤트 루프에 제어를 양보해야 한다.

### 배칭과 완전 비동기 (원문 의도 유지, 새 예제)

```python
import asyncio, aiohttp

# 완전 비동기: CPU 작업 중 I/O를 백그라운드로 진행
async def calculate(num_iter, difficulty):
    async with asyncio.TaskGroup() as tg:
        async with aiohttp.ClientSession() as session:
            for i in range(num_iter):
                result = do_cpu_task(difficulty)        # CPU 바운드
                tg.create_task(save_result(session, result))  # I/O는 큐에
                await asyncio.sleep(0)   # 이벤트 루프에 제어 양보 (필수!)
```

---

## Summary (핵심 정리)

- 외부 소스(DB·워커·데이터 서비스)와 통신하면 I/O 바운드가 되기 쉬우며, 동시성은 계산과 여러 I/O 작업을 인터리빙해 I/O와 CPU의 근본 차이를 활용한다.
- async/await와 asyncio·aiohttp로 동시 요청을 쌓으면 직렬 대비 수십 배 빨라지지만, 동시 요청 수·다운스트림 서비스 특성에 따라 효과가 달라지므로 자신의 애플리케이션에서 실험이 필수다.
- CPU+I/O 혼합 워크로드는 배칭(엔지니어링 부담이 적은 중간 해법)이나 완전 비동기(I/O를 CPU 뒤에 숨김, await asyncio.sleep(0)로 제어 양보)로 풀며, 완전한 CPU 가속은 다음 장의 multiprocessing이 필요하다.
