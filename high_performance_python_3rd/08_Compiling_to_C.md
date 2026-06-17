# 08. Compiling to C

## 챕터 개요 (3줄 요약)

- 좋은 알고리즘과 데이터 축소 후 더 빠르게 만드는 가장 쉬운 방법은 코드를 머신코드로 컴파일하는 것이며, Cython(AOT)·Numba(JIT)·PyPy(JIT) 중 코드 적응성과 팀 속도를 고려해 선택한다.
- 타입 정보를 주면 동적 타입 조회 오버헤드를 없애 C 수준으로 빨라지며, 수학적이고 루프가 많은 CPU 바운드 코드에서 1~2 자릿수 가속이 가능하다(I/O 바운드·외부 라이브러리 호출은 효과 적음).
- FFI(Foreign Function Interface, ctypes·cffi·f2py·CPython 확장·Rust)로 C/Fortran/Rust 코드를 Python에서 호출해 타겟 최적화나 기존 라이브러리를 활용할 수 있다.

---

## 1. JIT vs AOT & Why Types Help (JIT vs AOT, 타입이 빠른 이유)

> AOT(Cython)는 사용 전에 머신 특화 정적 라이브러리를 만들어 최고 속도를 주지만 수작업이 많고, JIT(Numba·PyPy)는 사용 시점에 컴파일해 노력은 적지만 "cold start" 문제가 있다.

- AOT는 numpy·scipy처럼 미리 컴파일된 라이브러리를 즉시 쓸 수 있고, JIT는 첫 실행이 느린 콜드 스타트가 있어 짧고 자주 실행되는 스크립트엔 불리하다.
- Python은 동적 타입이라 변수 타입을 매번 조회하고 어떤 함수 버전을 부를지 결정해야 해서 느리다(예: abs는 float와 complex에서 다르게 동작).
- 모든 기본 객체가 고수준 Python 객체(__hash__, __str__ 등)로 감싸져 있어 오버헤드가 크다.
- CPU 바운드 구간에선 변수 타입이 잘 안 바뀌므로, 타입을 미리 정해 정적 컴파일하면 참조 카운팅 없이 머신코드로 빠르게 계산할 수 있다.
- 컴파일이 효과적인 코드는 수학적이고 같은 연산을 많이 반복하며 임시 객체를 많이 만드는 루프다.

### Performance effort tradeoff (개념도)

```
  reward
    ^
    |    *  profiling -> algorithmic fix (big win)
    |       *  compiling (good win)
    |          *   *  more tuning (diminishing returns)
    +---------------------------> effort
```

---

## 2. Cython (타입 주석으로 C 컴파일)

> Cython은 타입 주석된 Python을 컴파일된 확장 모듈로 변환하며, .pyx 파일 + setup.py로 빌드하고 cdef로 C 타입을 선언해 가속한다.

- 빌드는 .pyx(컴파일 대상)·setup.py(빌드 지시)·호출 Python 코드 세 파일로 시작하며, `python setup.py build_ext --inplace`로 .so를 만든다.
- pyximport.install()을 쓰면 setup.py 없이 .pyx를 자동 컴파일할 수 있다.
- `cython -a`로 HTML 주석을 생성해, 음영(Python 가상머신 호출)이 많은 라인을 찾아 집중 최적화한다.
- cdef로 int·unsigned int·double complex 타입을 선언하면 타이트한 내부 루프가 Python 가상머신 호출 없이 C로 실행되어 줄리아 예제가 6s → 0.43s(약 13배)로 빨라진다.
- abs(complex) 대신 `z.real*z.real + z.imag*z.imag < 4`로 강도 감소(strength reduction)하면 0.23s(약 26배)가 된다.
- 자주 역참조하는 루프에선 boundscheck=False로 경계 검사를 끌 수 있다(외부 루프라 큰 이득은 아님).

### Cython 타입 주석 (원문 의도 유지, 새 예제)

```cython
# fastfn.pyx
def mandelbrot_escape(int maxiter, double complex c):
    cdef unsigned int n = 0
    cdef double complex z = 0
    # abs 대신 제곱 비교로 sqrt 회피 (강도 감소)
    while n < maxiter and (z.real * z.real + z.imag * z.imag) < 4:
        z = z * z + c
        n += 1
    return n
```

---

## 3. Cython + numpy + OpenMP (배열과 병렬화)

> numpy 배열은 연속 메모리에 저장되어 memoryview 버퍼 인터페이스로 C 속도 접근이 가능하며, OpenMP의 prange로 embarrassingly parallel 문제를 멀티코어로 돌릴 수 있다.

- list는 역참조마다 오버헤드가 있지만, array/numpy는 연속 블록이라 오프셋으로 다음 항목 주소를 C에서 바로 계산한다.
- `double complex[:] zs` 같은 memoryview 주석으로 버퍼 프로토콜 객체에 저수준 접근하며, C 라이브러리와 메모리 공유도 쉽다.
- Cython 주석 없이 numpy를 개별 역참조하면 오히려 순수 Python 리스트보다 느리다(8s vs 6s) — 컴파일로 오버헤드 제거.
- OpenMP는 `prange`와 `-fopenmp` 컴파일 플래그로 추가하며, `with nogil:` 블록에서 GIL을 풀어 병렬 실행한다(이때 Python 객체 조작 금지).
- schedule="guided"는 작업 시간이 가변적일 때 동적으로 청크를 분배해 idle 스레드를 줄여 0.03s까지 단축한다.

### prange로 OpenMP 병렬화 (원문 의도 유지, 새 예제)

```cython
from cython.parallel import prange
import numpy as np
cimport numpy as np

def calc(int maxiter, double complex[:] cs):
    cdef unsigned int i, length = len(cs)
    cdef double complex z, c
    cdef int[:] out = np.empty(length, dtype=np.int32)
    with nogil:                                   # GIL 해제
        for i in prange(length, schedule="guided"):
            z = 0; c = cs[i]; out[i] = 0
            while out[i] < maxiter and (z.real*z.real + z.imag*z.imag) < 4:
                z = z * z + c
                out[i] += 1
    return out
```

---

## 4. Numba & PyPy (JIT 컴파일러)

> Numba는 @jit 데코레이터로 numpy 코드를 LLVM으로 JIT 컴파일하고, PyPy는 트레이싱 JIT를 내장한 CPython 대체 인터프리터로 코드 변경 없이 가속한다.

- Numba는 @jit만 붙이면 줄리아 numpy 버전이 8s → 0.89s(첫 실행), 0.39s(두 번째 이후, 컴파일 캐시)로 빨라진다.
- @jit(parallel=True) + prange로 0.05s까지 단축하며, inspect_types()로 추론된 타입과 중간 표현을 디버깅할 수 있다.
- Numba는 작은(<10줄) 순수 numpy 함수에 잘 맞으며 GPU 코드도 생성하지만 외부 C 라이브러리는 못 묶는다.
- PyPy는 줄리아 순수 Python을 변경 없이 5.8s → 0.9s, 수학 확장 시 0.2s로 돌려 Cython 수준에 근접한다.
- PyPy는 mark-and-sweep GC라 파일 flush 등 참조 카운팅 의존 코드가 다르게 동작할 수 있어 with 컨텍스트 매니저 사용을 권장한다.
- PyPy는 numpy 같은 C 바인딩에 4~6배 오버헤드(cpyext/HPy)가 있어 numpy 多 사용 시 느려지고, RAM을 더 쓸 수 있다.

### Numba JIT (원문 의도 유지, 새 예제)

```python
from numba import jit, prange

@jit(parallel=True)                  # 두 번째 실행부터 컴파일 캐시로 빠름
def escape_counts(cs, maxiter, out):
    for i in prange(len(cs)):        # 자동 병렬화
        z = 0; c = cs[i]; n = 0
        while n < maxiter and (z.real*z.real + z.imag*z.imag) < 4:
            z = z * z + c
            n += 1
        out[i] = n
```

---

## 5. Choosing & Foreign Function Interfaces (선택 기준과 FFI)

> 순수 Python엔 PyPy, numpy 코드엔 Numba가 첫 선택이며, 가장 넓은 문제엔 Cython이 최선이다; 자동 방법이 부족하면 FFI로 C/Fortran/Rust 코드를 직접 호출한다.

- 비교: Cython(성숙·널리 쓰임·C 지식 필요·OpenMP), Numba(코드 변경 거의 없음·neuere), PyPy(numpy 외 강력·코드 변경 없음).
- 손으로 쓴 C가 항상 빠르다는 가정은 위험하며(32비트 float 실수 사례), 항상 벤치마크로 결정해야 한다.
- 기타 옵션: Pythran(과학자용 AOT), Transonic(통합 인터페이스), Shed Skin, PyCUDA/PyOpenCL, Nuitka.
- ctypes는 표준 라이브러리 FFI지만 타입 캐스팅·시그니처를 수동으로 다뤄야 해 복잡하고 깨지기 쉽다.
- cffi는 C 파서를 내장해 헤더 정의를 cdef에 넣으면 자동 처리하고, verify로 인라인 C를 동적 컴파일할 수 있어 ctypes보다 간편하다.
- f2py는 numpy에 포함되어 Fortran 코드를 !f2py 주석으로 쉽게 가져오며(LAPACK/BLAS 등), 단 column-major 순서(order='F')에 주의한다.
- CPython 확장(C)은 이식성은 높지만 매우 장황하고 유지보수가 어려워 최후의 수단이며, Rust(PyO3+maturin)는 메모리·스레드 안전성과 현대적 툴링으로 좋은 대안이다.

### cffi로 C 라이브러리 호출 (원문 의도 유지, 새 예제)

```python
from cffi import FFI

ffi = FFI()
ffi.cdef("double dot(double *a, double *b, int n);")   # 시그니처만 선언
lib = ffi.dlopen("./libmath.so")

def dot(a_np, b_np):
    pa = ffi.cast("double*", a_np.ctypes.data)
    pb = ffi.cast("double*", b_np.ctypes.data)
    return lib.dot(pa, pb, len(a_np))
```

---

## Summary (핵심 정리)

- 컴파일은 CPU 명령어 수를 줄여 효율을 높이는 방법으로, 수학적·루프 많은 CPU 바운드 코드에서 1~2 자릿수 가속을 주지만 I/O 바운드나 이미 벡터화된 numpy 호출엔 효과가 적다.
- 순수 Python엔 PyPy, numpy엔 Numba가 손쉬운 첫 선택이고, 타입 주석과 OpenMP 지원이 필요한 가장 넓은 문제엔 Cython이 최선이며, 모든 선택은 벤치마크로 검증해야 한다.
- 자동 컴파일이 부족하거나 기존 C/Fortran/Rust 라이브러리·언어 기능이 필요하면 FFI(ctypes·cffi·f2py·CPython 확장·Rust)로 다른 언어 코드를 호출하되, cffi/f2py/Rust가 ctypes/CPython-C보다 간편하고 안전하다.
