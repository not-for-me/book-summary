# 11. Performance

## 챕터 개요 (3줄 요약)
- 성능은 CPU·처리량·지연·메모리 등 다양한 지표로 측정되며, 어떤 지표가 중요한지는 문제 영역에 따라 다르다.
- 파이썬은 고성능 언어로 여겨지지 않지만, 프로파일링·마이크로벤치마크·네이티브 통합으로 놀라운 성능을 끌어낼 수 있다.
- 직관 대신 측정(cProfile, timeit)으로 최적화하고, 필요시 ctypes·C 확장·동적 임포트·zero-copy 기법을 활용한다.

---

## Item 92: 최적화 전에 프로파일링하라 (Profile Before Optimizing)
> 느림의 원인은 종종 불명확하므로 직관 대신 직접 측정해야 한다.

- 파이썬의 동적 특성 때문에 빠를 거라 여긴 연산(속성 접근, 함수 호출)이 느릴 수 있다.
- 순수 파이썬 `profile`보다 C 확장 `cProfile`이 오버헤드가 적어 정확하다.
- `Profile.runcall`로 함수 호출 트리를 격리해 프로파일링한다.
- `Stats`로 ncalls·tottime·cumtime 등을 정렬·출력한다.
- `print_callers`/`print_callees`로 공통 유틸 함수의 호출 관계를 파악한다.

```python
from cProfile import Profile
from pstats import Stats

profiler = Profile()
profiler.runcall(test)
stats = Stats(profiler)
stats.sort_stats("cumulative")
stats.print_stats()
```

## Item 93: timeit 마이크로벤치마크로 성능 핵심 코드를 최적화하라 (Optimize with timeit Microbenchmarks)
> timeit은 작은 코드 조각의 성능을 정밀 측정해 여러 구현을 과학적으로 비교한다.

- `timeit.timeit`은 코드 조각을 N번 반복한 총 시간을 반환한다.
- 노이즈 보정을 위해 충분히 큰 반복 횟수를 쓰고 반복당 시간으로 정규화한다.
- `setup` 인자로 초기화 시간을 측정에서 제외한다.
- `globals=globals()`로 외부 이름을 참조한다.
- `python -m timeit` CLI로 빠르게 성능을 조사한다(예: set이 list보다 1000배 빠른 멤버십 검사).

```python
import timeit
count = 1_000_000
delay = timeit.timeit(stmt="probe in numbers",
                      setup="numbers = set(range(10_000)); probe = 7_777",
                      globals=globals(), number=count)
print(f"{delay/count*1e9:.2f} nanoseconds")
```

## Item 94: 파이썬을 다른 언어로 교체할 시점과 방법을 알라 (Know When and How to Replace Python)
> 전면 재작성 전에 프로파일링·알고리즘 개선·네이티브 통합 등 모든 최적화를 먼저 시도한다.

- 임계 지연·시작 시간·특수 아키텍처 등은 다른 언어가 적합할 수 있다.
- 먼저 프로파일링·마이크로벤치마크로 진짜 원인을 찾는다.
- 핵심 알고리즘·자료구조 교체로 큰 효과를 볼 수 있다.
- 커널 함수 같은 타이트 루프는 명확한 인터페이스(seam)를 제공해 가속하기 좋다.
- `ctypes`, C 확장, NumPy, Numba, Cython, Mypyc, CFFI, SWIG 등 도구를 고려한다.

```python
# 타이트 루프는 네이티브로 가속하기 좋은 seam
def dot_product(a, b):
    result = 0
    for i, j in zip(a, b):
        result += i * j
    return result
```

## Item 95: 네이티브 라이브러리 빠른 통합엔 ctypes를 고려하라 (Consider ctypes to Rapidly Integrate)
> ctypes는 빌드 복잡도 없이 네이티브 라이브러리 함수를 빠르게 호출하게 한다.

- `ctypes.cdll.LoadLibrary`로 공유 라이브러리를 로드한다.
- `restype`/`argtypes`를 정확히 지정해야 암묵적 변환 오류를 막는다.
- C 확장과 달리 자동 메모리 관리·GIL 자동 해제·별도 빌드가 장점이다.
- 단, C가 표현 가능한 타입에 제한되어 이터레이터·덕타이핑을 잃는다.
- 잘못 쓰면 메모리 손상이 일어나므로 반드시 단위 테스트를 먼저 작성한다.

```python
import ctypes
my_library = ctypes.cdll.LoadLibrary(library_path)
my_library.dot_product.restype = ctypes.c_double
my_library.dot_product.argtypes = (ctypes.c_int, vector_ptr, vector_ptr)
```

## Item 96: 성능과 사용성 극대화엔 확장 모듈을 고려하라 (Consider Extension Modules)
> C 확장 모듈은 네이티브 속도로 실행되며 Python API의 모든 강력한 기능을 쓸 수 있다.

- 확장 모듈은 C로 작성되어 객체지향·덕타이핑·참조 카운팅 등을 활용한다.
- ctypes보다 복잡하지만 Pythonic한 인터페이스를 제공한다.
- 메모리 관리·에러 전파 등 Python API의 특이점이 어렵다.
- `PyObject_GetIter`, `PyNumber_Multiply` 등 프로토콜 API로 임의 이터러블·숫자 타입을 지원한다.
- 가장 큰 가치는 단순 C로 재현하기 어려운 Python 프로토콜·내장 타입 활용에 있다.

```c
PyObject *dot_product(PyObject *self, PyObject *args) {
    PyObject *left_iter = PyObject_GetIter(left);
    while (1) {
        left_item = PyIter_Next(left_iter);
        multiplied = PyNumber_Multiply(left_item, right_item);
        result = PyNumber_Add(result, multiplied);
    }
}
```

## Item 97: 시작 시간 개선엔 사전컴파일 바이트코드와 파일시스템 캐시를 활용하라 (Rely on Precompiled Bytecode and File System Caching)
> 바이트코드 사전 생성 + OS 메모리 캐시가 시작 시간을 최소화한다.

- CPython은 소스를 바이트코드로 컴파일 후 가상 머신에서 실행한다.
- 바이트코드는 `__pycache__`의 `.pyc`로 디스크에 캐시된다.
- 모듈은 첫 로드 후 메모리에 캐시되어 재사용된다.
- OS 파일시스템 캐시도 소스 I/O를 단축해 성능에 영향을 준다.
- `compileall`로 바이트코드를 미리 생성하고 메모리에 캐시되게 하면 가장 빠르다.

```bash
python3 -m compileall django   # 바이트코드 사전 생성
# 바이트코드 + 소스가 메모리 캐시에 있을 때 시작이 가장 빠름
```

## Item 98: 동적 임포트로 모듈을 지연 로딩해 시작 시간을 줄여라 (Lazy-Load Modules with Dynamic Imports)
> 무거운 의존성 초기화를 실제 사용 시점까지 지연시켜 시작 지연을 줄인다.

- 모든 모듈을 최상단에서 임포트하면 사용하지 않는 기능도 초기화 비용을 낸다.
- `-X importtime` 플래그로 모듈 로딩 시간을 진단한다.
- 함수 내부에서 import해 의존성 초기화를 지연시킨다.
- 동적 임포트 오버헤드는 약 50ns로, 웹 요청당 한 번이면 충분히 저렴하다.
- 콜드 스타트 지연이 큰 웹 핸들러에도 적용 가능하다.

```python
def main():
    args = parser.PARSER.parse_args()
    if args.command == "enhance":
        import enhance   # 지연 로딩
        enhance.do_enhance(args.file, args.amount)
```

## Item 99: bytes와의 zero-copy 상호작용엔 memoryview와 bytearray를 고려하라 (Consider memoryview and bytearray for Zero-Copy)
> memoryview는 복사 없이 슬라이스하고, bytearray는 가변 버퍼로 zero-copy 수신을 가능케 한다.

- `bytes` 슬라이싱은 데이터를 복사해 CPU 시간을 소모한다.
- `memoryview`는 버퍼 프로토콜로 복사 없이(zero-copy) 슬라이스한다.
- `bytearray`는 가변 bytes로 임의 위치를 덮어쓸 수 있다.
- memoryview로 bytearray를 감싸 복사 없이 데이터를 끼워넣는다.
- `socket.recv_into` 등은 버퍼에 직접 받아 메모리 할당·복사를 피한다.

```python
video_array = bytearray(video_cache)
write_view = memoryview(video_array)
chunk = write_view[byte_offset : byte_offset + size]
socket.recv_into(chunk)   # zero-copy 수신
```

---

## Summary (핵심 정리)
- 직관이 아닌 cProfile·timeit으로 측정해 진짜 병목을 찾고, 알고리즘·자료구조 개선을 먼저 시도한다.
- 성능 한계 시 ctypes(빠른 통합)나 C 확장(Pythonic+네이티브 속도)으로 핵심부를 가속한다.
- 시작 시간은 사전컴파일 바이트코드·동적 임포트로, I/O 처리량은 memoryview·bytearray의 zero-copy로 극대화한다.
