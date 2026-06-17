# 06. Matrix and Vector Computation

## 챕터 개요 (3줄 요약)

- 확산 방정식(diffusion equation)을 예제로 순수 Python → numpy → numexpr → GPU(PyTorch) 순으로 최적화하며, CPU 레벨에서 무슨 일이 일어나는지 perf로 측정한다.
- Python 리스트는 데이터를 포인터로 저장해 메모리 단편화(fragmentation)와 캐시 미스(cache-miss)를 유발하므로, 연속 메모리에 벡터화(vectorization)를 지원하는 numpy가 핵심 해법이다.
- GPU는 대규모 병렬 선형대수에 탁월하지만 데이터 전송 비용과 분기(branching)에 취약하며, 모든 최적화는 가설→측정→재측정으로 반드시 검증해야 한다.

---

## 1. The Problem & Why Python Lists Fall Short (확산 문제와 리스트의 한계)

> 확산 방정식을 오일러 방법(Euler's method)과 유한차분(finite differences)으로 이산화해 초기 상태를 시간에 따라 진화시키는 CPU 바운드 문제를 푼다.

- 확산(diffusion)은 유체를 균일하게 섞는 메커니즘으로, 열 방정식(heat equation)과 같은 형태다.
- 경계 조건(boundary condition)은 고정(fixed) 또는 주기적(periodic, 모듈로로 wrap)으로 처리한다.
- 메모리 할당은 비싸므로, evolve마다 new_grid를 새로 만들지 않고 미리 할당해 재사용하면 약 16% 빨라진다(루프 밖으로 불변 계산 빼기와 같은 원리).
- Python 리스트는 실제 데이터가 아닌 포인터를 저장해, grid[5][2] 접근에 다중 룩업이 필요하고 데이터가 메모리에 흩어진다(단편화).
- 단편화는 폰 노이만 병목(von Neumann bottleneck) — CPU와 메모리 간 제한된 대역폭 — 을 악화시켜 벡터화를 막는다.

### 메모리 재사용으로 할당 줄이기 (원문 의도 유지, 새 예제)

```python
# 나쁨: 매 호출마다 새 grid 할당
def evolve(grid, dt, D=1.0):
    new_grid = [[0.0] * M for _ in range(N)]   # 비싼 할당
    # ... 계산 후 new_grid 반환
    return new_grid

# 좋음: 미리 할당한 out을 재사용하고 swap
def evolve(grid, dt, out, D=1.0):
    for i in range(N):
        for j in range(M):
            out[i][j] = grid[i][j] + D * dt * laplacian_cell(grid, i, j)

# 호출부에서 두 grid를 swap (참조만 교환 → 매우 저렴)
grid, next_grid = next_grid, grid
```

---

## 2. Understanding perf (perf로 CPU 측정)

> Linux의 perf 도구는 CPU가 코드를 얼마나 효율적으로 실행하는지(캐시 미스, 페이지 폴트, 분기 예측 실패 등) 깊은 통찰을 준다.

- instructions/cycles 비율(insn per cycle)은 파이프라이닝과 벡터화가 얼마나 잘 되는지 보여준다.
- cache-references는 L1/L2 캐시 참조, cache-misses는 RAM에서 가져와야 했던 경우로, 순차 접근은 미스가 적다.
- page-faults(minor/major)는 지연 할당(lazy allocation) 때문에 새 메모리를 처음 접근할 때 커널이 개입하는 비싼 작업이다.
- branch-misses는 if 등 분기 예측이 틀린 경우로, 정렬된 리스트가 비정렬보다 빠른 이유가 되기도 한다.
- cs(context switch)와 migrations는 프로그램이 멈추거나 다른 코어로 옮겨지는 것으로, 고성능 수치 계산에선 베어메탈·단일 애플리케이션 실행이 필요할 수 있다.

### Memory hierarchy & von Neumann bottleneck (개념도)

```
   CPU registers
       |  (fastest)
   L1 / L2 / L3 cache   <- prefetch from RAM in contiguous chunks
       |
      RAM               <- fragmented list -> many small transfers (slow)
   contiguous array -> one bulk transfer -> vectorization possible
```

---

## 3. Enter numpy (numpy로 벡터화)

> numpy는 데이터를 연속 메모리에 저장하고 벡터화 연산을 지원하므로, 명시적 루프 없이 청크 단위로 계산해 순수 Python보다 훨씬 빠르다.

- numpy 배열은 동종(homogeneous) 저수준 숫자 타입으로 연속 저장되어 캐시 지역성(locality)과 벡터화를 얻는다.
- norm-squared 예제에서 numpy는 순수 Python보다 약 50배, `numpy.dot`은 중간 벡터 저장 없이 가장 빠르다.
- array 모듈은 연속 저장은 되지만 Python이 벡터화 바이트코드를 모르고, 인덱싱마다 변환 오버헤드가 있어 math엔 부적합하다.
- 확산 코드에 numpy의 `roll`(모듈로 인덱싱의 벡터화)을 쓰면 코드가 짧아지고 순수 Python 대비 약 75배 빨라진다.
- 속도 향상은 명령어 수가 약 47.8배 줄고, 남은 명령어가 여러 CPU 코어(약 1.6개)를 쓴 결과다 — "가장 빠른 코드는 실행하지 않는 코드".

### numpy 벡터화 (원문 의도 유지, 새 예제)

```python
import numpy as np

# 명시적 루프 없이 청크 단위 계산
def laplacian(grid):
    return (np.roll(grid, +1, 0) + np.roll(grid, -1, 0) +
            np.roll(grid, +1, 1) + np.roll(grid, -1, 1) - 4 * grid)

def evolve(grid, dt, D=1):
    return grid + dt * D * laplacian(grid)

# norm-squared는 dot으로 중간 배열 없이
norm_sq = np.dot(vector, vector)
```

---

## 4. In-Place Operations & numexpr (인플레이스 연산과 numexpr)

> 인플레이스 연산(+=, *=)은 입력을 출력으로 재사용해 새 할당을 피하며, numexpr은 벡터 표현식 전체를 컴파일해 캐시 미스와 임시 공간을 최소화한다.

- 할당은 캐시 미스보다 비싼데, OS 커널에 요청해 공간을 예약해야 하기 때문이다.
- 인플레이스 연산은 배열의 id(메모리 주소)를 바꾸지 않으며, 큰 배열에서 약 27% 빨라진다(단 캐시에 들어가는 작은 배열은 out-of-place가 더 빠름).
- 인플레이스로 바꾸면 가독성이 떨어지므로(A = A*B+C가 여러 줄로) 충분한 속도 이득이 있을 때만 정당화된다.
- np.roll은 새 배열을 할당하므로, 문제에 특화된 `roll_add` 함수를 만들면 할당과 분기를 줄여 약 7% 더 빨라진다(반드시 docstring·테스트 동반).
- numexpr은 전체 표현식을 문자열로 받아 fused multiply-add·멀티코어·OpenMP를 활용하며, 큰 그리드(1024² 이상)에서 numpy를 능가한다.
- 작은 그리드에선 numexpr의 컴파일·코어 관리 오버헤드가 이득을 상쇄한다.

### 인플레이스 + numexpr (원문 의도 유지, 새 예제)

```python
import numpy as np
from numexpr import evaluate

def laplacian(grid, out):
    np.copyto(out, grid)
    out *= -4
    out += np.roll(grid, +1, 0); out += np.roll(grid, -1, 0)
    out += np.roll(grid, +1, 1); out += np.roll(grid, -1, 1)

def evolve(grid, dt, out, D=1):
    laplacian(grid, out)
    # 한 표현식으로 컴파일 → 캐시 친화적, 임시 배열 없음
    evaluate("out * D * dt + grid", out=out)
```

---

## 5. GPUs with PyTorch (PyTorch로 GPU 활용)

> GPU는 클럭은 느리지만 수천 개의 코어로 병렬화 가능한 문제(선형대수)를 극적으로 가속하며, PyTorch의 동적 그래프(dynamic graph)는 numpy와 유사한 직관적 API를 제공한다.

- GPU는 코어 수가 압도적(예: RTX 2080 TI 4,352코어)이라 확산 문제에서 512²는 약 10.5배, 12,288²는 약 63.6배 빨라진다.
- PyTorch는 `numpy→torch`로 import만 바꾸고 `.to(device='cuda')`로 데이터를 옮기면 GPU 코드로 자동 컴파일된다(인플레이스는 `_` 접미사).
- GPU 타이밍 시 `CUDA_LAUNCH_BLOCKING=1`로 비동기 실행을 막아 실제 계산 시간을 측정해야 한다.
- GPU는 저정밀도(float32/16)에서 데이터 처리량이 2~4배 늘어 더 빠르지만, CPU는 16비트 명령이 없으면 오히려 느려진다.
- Laplacian을 합성곱(convolution)으로 재구성하면 GPU 전용 최적화 커널을 써서 큰 그리드에서 약 4배 추가 가속된다.
- 최대 병목은 CPU↔GPU 데이터 전송이며, `pin_memory()`로 page-locked 영역을 쓰면 비동기 전송으로 완화할 수 있다.
- AMP(Automatic Mixed-Precision)는 연산별로 최적 정밀도를 자동 선택하고, 모델 양자화(quantization)·분산(FSDP/TP/PP)·JIT 등 딥러닝 전용 도구가 있다.

### PyTorch GPU 확산 (원문 의도 유지, 새 예제)

```python
import torch

def laplacian(grid, out):
    out.copy_(grid)
    out *= -4
    out += torch.roll(grid, +1, 0); out += torch.roll(grid, -1, 0)
    out += torch.roll(grid, +1, 1); out += torch.roll(grid, -1, 1)

grid = torch.zeros((512, 512)).to(device='cuda')   # GPU로 이동
scratch = torch.zeros((512, 512)).to(device='cuda')

# 저정밀도로 추가 가속
torch.set_default_dtype(torch.float16)

# AMP로 연산별 정밀도 자동 선택
with torch.amp.autocast(device_type="cuda"):
    result = torch.mm(a, b)   # 내부적으로 float16 사용
```

---

## 6. Verify Optimizations & Lessons (최적화 검증과 교훈)

> scipy의 laplace 필터처럼 "빠를 것 같은" 라이브러리도 일반화 코드라 명령어가 2배 이상 많아 실제로는 더 느릴 수 있으므로, 항상 측정으로 검증해야 한다.

- scipy laplace는 모든 경계 조건을 처리하려 분기가 많아 벡터화·파이프라이닝이 막혀 기대만큼 빠르지 않았다.
- 초기화(할당·설정 읽기·사전계산)는 한 번만 하도록 앞쪽에 모으면 파이프라이닝과 캐시 활용에 유리하다.
- 데이터 지역성과 데이터를 CPU에 빠르게 전달하는 것이 핵심이며, L3 캐시가 그리드로 가득 차면 속도 향상이 포화된다.
- 외부 라이브러리(저수준 언어로 작성)는 매우 빠르지만, 최적화가 다른 컴퓨터로 일반화되는지와 가독성을 고려해야 한다.
- 가설을 먼저 세우고 벤치마크로 검증하는 것이 최적화의 정량적 나침반이다.

---

## Summary (핵심 정리)

- 벡터 계산 최적화의 두 축은 데이터를 CPU에 빠르게 전달하는 것(데이터 지역성·인플레이스 연산)과 CPU가 할 일을 줄이는 것(벡터화·특화 코드)이며, numpy는 순수 Python 대비 약 75배 가속을 준다.
- perf로 캐시 미스·페이지 폴트·분기 예측을 측정해 병목을 진단하고, numexpr·커스텀 함수·GPU(PyTorch)로 단계적으로 가속하되 그리드 크기에 따라 효과가 달라짐을 유의해야 한다.
- GPU는 대규모 병렬 선형대수에 탁월하지만 데이터 전송·분기·메모리 한계에 취약하므로, 메모리 적합성·벡터화 가능성·전송량·API 지원을 따져 사용하고, 모든 "최적화"는 가설→측정→재측정으로 반드시 검증해야 한다.
