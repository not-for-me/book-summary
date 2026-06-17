# 01. Understanding Performant Python

## 챕터 개요 (3줄 요약)

- 고성능 프로그래밍은 연산 오버헤드를 줄이거나 더 적합한 알고리즘으로 각 연산을 더 의미있게 만드는 작업이며, 이를 위해 CPU·메모리·통신(버스)이라는 컴퓨터의 근본 구조를 이해해야 한다.
- Python은 하드웨어를 추상화해 개발 속도를 높여주지만, GIL(Global Interpreter Lock)·동적 타입·비컴파일·메모리 단편화 때문에 성능 비용이 발생하며, 이는 NumPy·Cython·multiprocessing 등으로 완화할 수 있다.
- 진정한 고성능 프로그래머는 코드 속도만이 아니라 문서화·테스트·구조·팀 협업을 통해 장기적 팀 속도(velocity)를 최적화하는 사람이다.

---

## 1. The Fundamental Computer System (컴퓨터의 근본 구조)

> 컴퓨터는 연산 유닛(computing unit), 메모리 유닛(memory unit), 그리고 이들을 잇는 연결(connection)이라는 세 가지 기본 부품으로 단순화할 수 있다.

- CPU(Central Processing Unit)는 가장 흔한 연산 유닛이고, GPU(Graphics Processing Unit)·TPU·IPU·FPGA 등은 본질적 병렬성(intrinsic parallelism) 덕분에 수치 연산용 보조 유닛으로 부상하고 있다.
- 연산 유닛의 핵심 지표는 사이클당 명령어 수 IPC(Instructions Per Cycle)와 클럭 속도(clock speed)이며, 이 둘은 서로 경쟁 관계에 있다.
- 벡터화(vectorization)는 여러 데이터를 한 번에 처리하는 SIMD(Single Instruction, Multiple Data) 명령으로 처리량을 크게 늘린다.
- 트랜지스터 미세화의 물리적 한계로 클럭·IPC가 정체되면서, 제조사는 하이퍼스레딩(hyperthreading), 비순차 실행(out-of-order execution), 멀티코어(multicore)로 속도를 확보한다.
- 암달의 법칙(Amdahl's law)에 따르면 병렬화 가능한 부분을 아무리 늘려도, 반드시 직렬(serial)로 실행되어야 하는 부분이 최대 속도 향상의 한계가 된다.
- 메모리 유닛은 읽기/쓰기 속도와 용량이 반비례하므로(HDD → SSD → RAM → L1/L2 cache), 시스템은 계층적(tiered) 접근을 사용한다.
- 버스(bus)의 속도는 한 번에 옮기는 데이터 양(bus width)과 초당 전송 횟수(bus frequency)로 결정되며, GPU는 느린 PCI 버스로 인해 데이터 전송 비용이 큰 단점을 가진다.

### Memory Hierarchy (메모리 계층)

```
        speed (fast)                          capacity (large)
  +----------------+
  |  L1 / L2 cache |  <- tens of MB, fastest
  +----------------+
  |      RAM       |  <- tens of GB, fast random access
  +----------------+
  |      SSD       |  <- ~1 TB
  +----------------+
  |      HDD       |  <- ~20 TB, slow, persistent
  +----------------+
   CPU <--frontside bus--> cache <--> RAM <--external/PCI bus--> GPU / disk / network
```

---

## 2. Idealized Computing Versus the Python Virtual Machine (이상적 연산 vs 파이썬 가상머신)

> 이상적인 컴퓨터는 데이터를 최소한으로 이동시키고 캐시를 가득 채워 벡터화로 연산하지만, Python 가상머신은 추상화 대가로 이 최적화 기회들을 대부분 잃는다.

- 핵심 최적화 원리는 "데이터를 필요한 곳에 두고 최소한으로 이동"하는 것이며, 데이터 이동 비용을 "heavy data"라 부른다.
- 이상적으로는 `number`와 여러 개의 `i` 값을 한 번에 CPU 캐시로 보내 벡터화 연산을 수행하지만, 순수 Python에서는 루프가 한 번에 한 `i`씩만 처리된다.
- Python은 가비지 컬렉션(garbage collection)으로 인한 메모리 단편화(memory fragmentation) 때문에 객체가 캐시에 최적으로 배치되지 못한다.
- 동적 타입(dynamic typing)과 비컴파일 특성 때문에 컴파일러 수준의 최적화가 어려우며, 이는 Cython이나 JIT(Just-In-Time) 컴파일러로 완화된다.
- GIL(Global Interpreter Lock)은 한 번에 하나의 명령어만 실행하게 하여 멀티스레드 병렬화 이점을 무력화하므로, multiprocessing이나 Cython으로 우회한다.
- 같은 O(n)이라도 조기 종료(early termination) 여부에 따라 실행 속도가 달라지며, 어떤 함수가 더 빠른지는 데이터 특성에 따라 달라지므로 프로파일링이 필수다.

### 벡터화 의도를 보여주는 예제 (원문 의도 유지, 새 예제로 재작성)

```python
import math

# 순수 Python: 한 번에 i 하나씩 (스칼라 처리)
def is_prime_scalar(n: int) -> bool:
    if n < 2:
        return False
    limit = int(math.isqrt(n))
    for divisor in range(2, limit + 1):
        if n % divisor == 0:
            return False
    return True

# NumPy 벡터화: 여러 divisor를 한 번에 검사 (SIMD 활용 의도)
import numpy as np

def is_prime_vectorized(n: int) -> bool:
    if n < 2:
        return False
    divisors = np.arange(2, int(math.isqrt(n)) + 1)
    # 한 연산으로 모든 나머지를 동시에 계산
    return not np.any(n % divisors == 0)
```

---

## 3. So Why Use Python? (왜 파이썬인가)

> Python은 표현력이 높고 배우기 쉬우며, C/Fortran으로 작성된 고성능 라이브러리를 감싸기 때문에 올바르게 쓰면 C에 준하는 속도를 낼 수 있다.

- scikit-learn은 LIBLINEAR·LIBSVM(C)을, NumPy는 BLAS 등 C/Fortran 라이브러리를 감싸므로 적절히 사용하면 빠르다.
- "배터리 포함(batteries included)" 철학으로 io·array·math·sqlite3·collections·asyncio 등 강력한 표준 라이브러리를 기본 제공한다.
- 외부 생태계로 NumPy·SciPy·pandas·polars·scikit-learn·PyTorch·TensorFlow·spaCy 등 분야별 핵심 라이브러리가 풍부하다.
- 빠른 프로토타이핑(fast prototyping)이 Python의 핵심 강점이며, 아이디어 검증을 신속하게 할 수 있다.
- 단 Cython 같은 도구로 속도를 얻으면 유지보수 비용이 증가하므로 "시스템이 빨라지면 팀은 느려지는가?"를 항상 고민해야 한다.

---

## 4. How to Be a Highly Performant Programmer (고성능 프로그래머가 되는 법)

> 고성능 코드는 일부일 뿐이며, 좋은 구조·문서화·디버깅 용이성·공유 표준을 통한 팀 전체의 속도가 훨씬 더 중요하다.

- 개발 순서는 "동작하게(make it work) → 올바르게(make it right) → 빠르게(make it fast)"이며, 프로파일링은 마지막 단계다.
- 필수 요소는 문서화(README/NOTES), 좋은 구조, 테스트(pytest + coverage)이며, 코드에 물리면 반드시 테스트를 추가한다.
- TDD(Test-Driven Development)는 입출력을 미리 정의해 로직 구현을 단순하게 만들지만, 데이터를 아직 이해 못한 탐색적 연구에는 잘 맞지 않는다.
- 소스 컨트롤을 항상 사용하고 자주 커밋하며, PEP 8 표준과 black·flake8로 코드 품질을 자동 유지한다.
- 코드 블록이 아니라 팀을 최적화해야 하며, 매일 영향을 주는 1% 개선이 연 몇 시간짜리 문제의 100배 개선보다 가치 있을 수 있다.
- 원격/하이브리드 환경에서는 "문서화 우선(documentation first)" 문화와 정기적 체크인이 고립감을 줄이고 협업을 가능하게 한다.
- Jupyter Notebook은 시각적 소통엔 좋지만 게으름을 유발하므로, 긴 함수는 모듈로 추출해 테스트를 붙이고 assert로 검증한다.
- 번아웃 방지를 위해 호기심과 기쁨을 유지하고, 축하할 일들의 목록을 기록하는 것이 장기적 생산성에 중요하다.

### Make it work → right → fast (개발 흐름)

```
  [Make it work]   prototype, "one to throw away", plan first
        |
        v
  [Make it right]  test suite + docs + reproducibility
        |
        v
  [Make it fast]   profile -> compile / parallelize -> verify with tests
```

---

## 5. The Future of Python (파이썬의 미래)

> 출판 시점 기준 두 가지 큰 변화가 진행 중이다: GIL의 선택적 제거와 JIT 컴파일러 도입.

- PEP 703은 과학/AI 응용을 위한 GIL-free Python을 제안하며, 10~100개 스레드의 CPU 집약적 코드에서 병렬화 기회를 살리는 것이 목표다.
- NumPy·Pandas 등은 현재 GIL에 의존하는 C 코드를 갖고 있어 전환에 코드 조정이 필요하며, GIL-free 버전은 대략 2028년경 일반 제공이 기대된다.
- Python 3.13부터 "copy and patch" 방식의 JIT(Just-In-Time) 컴파일러가 CPython에 내장되며, 이는 Lua에서 처음 쓰인 설계다.
- 이 JIT는 LLVM으로 미리 만든 반쯤 컴파일된 "스텐실(stencil)"의 "구멍"을 런타임에 변수 메모리 주소로 채워 빠른 머신코드를 생성한다.
- JIT는 네이티브 Python(특히 수치 Python)에 효과가 크지만, 이미 C/Cython으로 컴파일된 Pandas·NumPy·SciPy 코드에는 영향이 적다.

---

## Summary (핵심 정리)

- 고성능 Python의 출발점은 CPU·메모리·버스라는 하드웨어 구조와, Python이 그것을 어떻게 추상화하는지(그리고 그 추상화가 만드는 GIL·메모리 단편화·동적 타입 비용)를 이해하는 것이다.
- 순수 Python의 성능 제약은 NumPy(벡터화), Cython/Numba/JIT(컴파일), multiprocessing(병렬화)으로 체계적으로 완화할 수 있으며, 어떤 최적화가 유효한지는 반드시 프로파일링으로 확인해야 한다.
- 진정한 성능은 코드 속도가 아니라 문서화·테스트·구조·팀 협업을 통한 장기적 팀 속도이며, "동작하게 → 올바르게 → 빠르게"의 순서를 지키는 것이 핵심이다.
