# 10. The multiprocessing Module

## 챕터 개요 (3줄 요약)

- CPython은 GIL 때문에 기본적으로 멀티코어를 쓰지 못하므로, multiprocessing 모듈로 각자 GIL을 가진 독립 프로세스를 띄워 CPU 바운드 문제를 병렬화한다.
- 상태 공유(state sharing)는 동기화 비용이 커서 embarrassingly parallel(공유 없음)이 가장 효율적이며, IPC(프로세스 간 통신)는 강력하지만 복잡하고 종종 단순 해법이 더 낫다.
- 큰 numpy 배열을 복사 없이 공유하고, 파일·변수 접근을 락(lock)으로 동기화해 데이터 손상을 막으며, Joblib은 병렬화와 캐싱을 간편하게 해준다.

---

## 1. Overview & Processes vs Threads (개요와 프로세스 vs 스레드)

> n개 코어로 최대 n배 가속이 가능하지만, 통신 오버헤드와 RAM 감소로 보통 3~4배에 그치며, 암달의 법칙(Amdahl's law)이 병렬화의 상한을 정한다.

- multiprocessing의 주요 구성요소: Process(포크된 자식 프로세스), Pool(워커 풀), Queue(FIFO 큐), Pipe(통신 채널), Manager(객체 공유), ctypes(원시 타입 공유), 동기화 프리미티브(Lock/Semaphore).
- Python 스레드는 OS 네이티브지만 GIL에 묶여 한 번에 하나만 Python 객체와 상호작용하므로 CPU 바운드엔 부적합하다.
- 프로세스는 각자 private 메모리와 GIL을 가져 경쟁 없이 병렬 실행되므로 CPU 바운드 가속의 가장 쉬운 방법이다.
- 공유 상태가 적으면(embarrassingly parallel) 프로세스를 늘려도 페널티가 적지만, 모두가 통신하면 오버헤드가 처리를 압도해 오히려 느려진다.
- 각 포크 프로세스는 10~20MB(라이브러리 많으면 수백 MB) RAM을 쓰므로 RAM 부족 시 디스크 스왑으로 이득이 사라진다.
- 기본적으로 하이퍼스레드를 추가 CPU로 보지만, 캐시 비친화적이라 numpy 외엔 큰 이득이 없다.

---

## 2. Estimating Pi & Joblib (몬테카를로 파이 추정과 Joblib)

> 단위 원에 무작위 다트를 던지는 몬테카를로 방법은 작업을 균등 분할할 수 있어 병렬화 학습에 이상적이며, 프로세스는 선형 가속을, 스레드는 GIL 때문에 가속 없음을 보여준다.

- 순수 Python 루프 버전은 스레드로 가속이 안 되지만(GIL), 2~8 프로세스에서 코어 수에 비례해 가속된다(8코어 이후 하이퍼스레드는 거의 무익).
- numpy 벡터화 버전은 연속 메모리·저수준 타입으로 순수 Python보다 약 16배 빠르고, 캐시 친화적이라 스레드에서도 약간의 가속을 얻는다(GIL 밖에서 동작).
- Joblib은 multiprocessing 개선판으로 Parallel 클래스와 delayed 데코레이터로 간단히 병렬화하며, Loky+cloudpickle로 대화형 함수도 피클링한다.
- Memory 캐시 데코레이터는 입력 인자 기반으로 결과를 디스크에 캐싱해 세션 간 유지하므로, 두 번째 실행이 즉시 완료된다(단 인자가 같으면 캐시 충돌 주의).
- 병렬 시스템에서 numpy 난수는 프로세스마다 `seed()`를 호출해야 하며, 안 하면 모든 포크가 동일 난수열을 생성한다.

### Joblib 병렬화와 캐싱 (원문 의도 유지, 새 예제)

```python
from joblib import Parallel, delayed, Memory

memory = Memory("/tmp/cache", verbose=0)

@memory.cache                          # 결과를 디스크에 캐싱 (세션 간 유지)
def estimate_block(n_samples, idx):    # idx로 캐시 충돌 방지
    import numpy as np
    np.random.seed()                   # 프로세스마다 시드 필수!
    xs, ys = np.random.uniform(0, 1, (2, n_samples))
    return int(np.sum(xs*xs + ys*ys <= 1))

counts = Parallel(n_jobs=8)(
    delayed(estimate_block)(50_000_000, i) for i in range(8)
)
pi = sum(counts) * 4 / (8 * 50_000_000)
```

---

## 3. Finding Primes — Pool & chunksize (소수 찾기와 작업 분배)

> 소수 판정은 위치마다 복잡도가 달라 부하 분산(load balancing)이 중요하며, chunksize로 통신 오버헤드와 자원 제어를 조율한다.

- 소수 판정 비용은 √n에 비례해 n이 커질수록 증가하고 예측 불가능하므로, 작업 복잡도가 가변적이다.
- chunksize가 작으면(=1) 유연하지만 통신 오버헤드가 크고, 키우면 오버헤드가 줄다가 어느 지점에서 자원 미활용으로 다시 나빠진다.
- 기본 동작(전체 작업/CPU 수)이 보통 최적이며, 확실한 이득이 있을 때만 조정한다.
- 청크 수를 물리 CPU 수의 배수로 맞춰야 자원을 완전 활용하며, 8코어에 9청크면 8개 후 1개가 따로 돌아 비효율적이다.
- 가변 런타임 작업은 작은 작업을 많이 만들거나(느린 작업 먼저 정렬), 기본 chunksize를 쓰는 것이 좋다.

### chunksize 조정 (개념)

```
  100,000개 작업, 8 CPU
  chunksize=1     -> 통신 오버헤드 큼 (병목)
  chunksize=기본  -> 작업/CPU = 균등 분배 (대개 최적)
  chunksize=50000 -> 2청크만 생성, 6 CPU idle (비효율)
```

---

## 4. Queues & IPC for Prime Verification (큐와 IPC로 소수 검증)

> Queue는 피클링 가능한 객체를 프로세스 간 전달하지만 피클링·동기화 오버헤드가 크고, IPC로 인수를 찾으면 조기 종료해 단일 CPU보다 빠르게 소수를 검증할 수 있다.

- multiprocessing.Queue는 비영속 FIFO 큐로, 가벼운 작업에선 통신 비용이 지배적이라 단일 프로세스보다 느릴 수 있다.
- poison pill(sentinel) 패턴으로 워커 종료를 신호하며, Thread로 비동기 작업 공급도 가능하지만 robust한 시스템엔 Kafka/ZeroMQ/Celery 같은 외부 큐를 권장한다.
- 큰 수 하나의 소수 검증은 인수 범위를 여러 CPU에 나누고, 한 CPU가 인수를 찾으면 공유 플래그로 조기 종료한다.
- 공유 플래그 방법별 속도: Manager.Value/Redis(느림, 통신 오버헤드) < RawValue/mmap(빠름).
- "Less Naive Pool"(짧은 인수 직렬 선검사 후 병렬)은 단순하면서도 강력한 벤치마크다.
- Redis는 언어 무관·네트워크 공유·외부 모니터링이 가능해 팀 협업에 유리하지만 통신 오버헤드가 있다.
- mmap(공유 메모리)은 동기화 없는 바이트 직접 접근이라 가장 빠르며, 루프 언롤링으로 더 최적화할 수 있다.

### mmap 공유 플래그로 조기 종료 (원문 의도 유지, 새 예제)

```python
import mmap

flag = mmap.mmap(-1, 1)   # 익명 1바이트 공유 메모리

def check_factor_range(n, lo, hi):
    for i in range(lo, hi, 2):
        if i % 5000 == 1:          # 주기적으로 플래그 확인
            flag.seek(0)
            if flag.read_byte() == 1:   # 다른 프로세스가 인수 발견
                return False
        if n % i == 0:
            flag.seek(0); flag.write_byte(1)   # 인수 발견 신호
            return False
    return True
```

---

## 5. Sharing numpy & Synchronizing Access (numpy 공유와 접근 동기화)

> 큰 numpy 배열을 복사 없이 프로세스 간 공유하면 RAM과 복사 시간을 절약하며, 파일·변수 공유 시 락이 없으면 데이터가 손상된다.

- `multiprocessing.Array(lock=False)`로 공유 바이트 블록을 만들고 `np.frombuffer`로 numpy 배열로 감싼 뒤 reshape하면, 여러 프로세스가 같은 메모리를 읽고 쓴다.
- 30.5GB 배열을 8프로세스가 복사 없이 공유해, pmap으로 보면 모두 같은 공유 메모리 블록을 가리킨다(BLAS/MKL 등 기존 라이브러리 우선 검토 권장).
- 락 없이 여러 프로세스가 파일을 읽고 쓰면 빈 파일에서 동시 시작·부분 쓰기 읽기로 카운트가 망가진다.
- fasteners의 `@interprocess_locked` 데코레이터로 파일 락을 걸면 한 번에 한 프로세스만 쓴다(Python 전용 락).
- `multiprocessing.Value`의 락은 동시 읽기/쓰기는 막지만 원자적 증가(atomic increment)는 보장하지 않아, 별도 Lock이 필요하다.
- RawValue는 락이 없어 더 빠르며, 동기화를 피하려면 플래그가 한 방향으로만 바뀌는 등 올바른 이유가 있어야 한다.

### 공유 numpy 배열과 Lock (원문 의도 유지, 새 예제)

```python
import multiprocessing, ctypes
import numpy as np

# 복사 없는 공유 배열
base = multiprocessing.Array(ctypes.c_double, 10000 * 400000, lock=False)
shared = np.frombuffer(base, dtype=ctypes.c_double).reshape(10000, 400000)

# 카운터 동기화: 원자적 증가를 위해 명시적 Lock 필요
def increment(value, lock, n):
    for _ in range(n):
        with lock:                # 컨텍스트 매니저로 락 (가독성)
            value.value += 1

value = multiprocessing.Value('i', 0)
lock = multiprocessing.Lock()
```

---

## Summary (핵심 정리)

- multiprocessing은 각자 GIL을 가진 독립 프로세스로 CPU 바운드 문제를 병렬화해 코어 수에 비례한 가속을 주지만, 통신 오버헤드와 암달의 법칙으로 상한이 있고 numpy 벡터화가 순수 Python보다 훨씬 빠르다.
- Queue와 IPC(Manager/Redis/RawValue/mmap)로 상태를 공유할 수 있으나 동기화 비용이 크므로, 종종 단순한 병렬 해법이나 더 빠른 컴퓨터 구입이 더 실용적이며, 외부 큐(Redis 등)는 디버깅·가시성에 유리하다.
- 큰 numpy 배열은 `Array`+`frombuffer`로 복사 없이 공유해 RAM을 절약하고, 파일·변수 공유 시 fasteners·Lock으로 반드시 동기화해 데이터 손상을 막되, 성능보다 팀이 이해하기 쉬운 코드를 우선해야 한다.
