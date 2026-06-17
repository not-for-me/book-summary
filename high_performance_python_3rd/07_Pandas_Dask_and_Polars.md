# 07. Pandas, Dask, and Polars

## 챕터 개요 (3줄 요약)

- Pandas는 이종(heterogeneous) 컬럼을 가진 테이블형 데이터를 다루는 사실상의 표준이며, apply(특히 raw=True)·중간 결과 리스트 구축·문자열 최적화로 빠르게 쓸 수 있다.
- Numba JIT 컴파일과 engine="numba"·병렬화를 결합하면 행 단위 연산을 최대 약 23배까지 가속할 수 있고, Dask·Swifter로 멀티코어·larger-than-RAM 처리가 가능하다.
- Polars는 Arrow 기반에 쿼리 최적화기(query optimizer)를 내장한 신생 경쟁자로, 동일 코드에서 Pandas보다 보통 2~10배 빠르다.

---

## 1. Pandas Internal Model (Pandas 내부 모델)

> Pandas는 NumPy와 Arrow 위에서 동종 컬럼을 이종 타입 테이블로 저장하며, 모든 연산을 즉시(eager) 실행하고 쿼리 계획(query planning)이 없어 임시 배열로 RAM을 많이 쓴다.

- 컬럼 연산은 임시 중간 배열을 만들어 현재 사용량의 3~5배 RAM을 쓸 수 있으며, 보통 10GB 미만 데이터에 적합하다.
- 같은 dtype 컬럼은 BlockManager로 묶여 행 단위 연산이 빨라지지만, 다른 블록을 가로지르는 슬라이스는 복사(copy)를 유발할 수 있다.
- 숫자 컬럼은 NumPy 데이터를 직접 참조하지만 문자열 컬럼은 메모리에 흩어진 Python str 리스트를 참조해 속도 차이가 생긴다.
- NumPy int64는 NaN을 모르지만 Pandas Int64(대문자 I)는 정수 배열 + Boolean NaN 마스크 두 컬럼을 써서 결측치를 표현한다.
- Pandas 2.x는 PyArrow 백엔드를 도입했는데, 문자열은 압축 표현으로 RAM을 크게 절약하지만 Numba는 Arrow를 컴파일하지 못한다(NumPy만 가능).
- Pandas 3.0의 Copy on Write 모드는 배경 복사를 줄여 메모리 압박과 실행 속도를 개선한다.

---

## 2. Applying a Function to Rows (행에 함수 적용하기)

> 행 단위 연산은 iloc·iterrows 같은 루프가 가장 느리고, apply(특히 raw=True)가 중간 Series 생성을 피해 가장 빠르고 가독성도 좋다.

- OLS(Ordinary Least Squares) 기울기 계산에서 scikit-learn의 LinearRegression은 데이터 검증(_validate_data, _preprocess_data)에 85% 시간을 써, 순수 numpy.linalg.lstsq보다 약 7배 느리다.
- 검증은 잘못된 데이터로 인한 디버깅을 막아주므로, 데이터 형식을 확신할 때만 안전망을 제거하고 단위 테스트로 보완해야 한다.
- 속도 순서: iloc(7.3s) < iterrows(7.9s) < apply(3.9s) < apply raw=True(3.1s).
- raw=True는 중간 Series를 만들지 않고 NumPy 배열에 직접 접근하며, Numba·Cython 컴파일을 가능하게 한다.

### 행 단위 적용 (원문 의도 유지, 새 예제)

```python
import numpy as np

def ols_slope_raw(row):           # row는 numpy 배열
    x = np.arange(row.shape[0])
    A = np.vstack((x, np.ones(row.shape[0]))).T
    m, c = np.linalg.lstsq(A, row, rcond=-1)[0]
    return m

# 느림: 루프
slopes = [ols_slope_raw(df.iloc[i].values) for i in range(len(df))]

# 빠름: apply + raw=True (중간 Series 생성 회피)
slopes = df.apply(ols_slope_raw, axis=1, raw=True)
```

---

## 3. Numba Compilation & Building Results (Numba 컴파일과 결과 구축)

> raw=True 변형을 Numba JIT로 컴파일하면 약 5배, engine="numba"와 병렬화를 더하면 순수 대비 약 23배까지 빨라진다.

- `jit(func, nopython=True)`로 컴파일하면 100,000행 처리가 3.1s → 0.66s로 빨라진다(첫 호출은 컴파일 지연 포함).
- `apply(..., engine="numba")` fast-path는 0.40s, `engine_kwargs={'parallel':True}`로 멀티코어까지 쓰면 0.13s가 된다.
- Numba는 NumPy만 컴파일하므로 PyArrow 배열엔 적용되지 않는다.
- concat을 반복 호출하면 매번 새 Series를 메모리에 만들어 점점 느려지므로(리스트 7s vs concat 14s), 중간 결과를 리스트에 모았다가 한 번에 Series/DataFrame으로 만들어야 한다.

### Numba 컴파일과 결과 구축 (원문 의도 유지, 새 예제)

```python
from numba import jit

fast_slope = jit(ols_slope_raw, nopython=True)
slopes = df.apply(fast_slope, axis=1, raw=True)

# 또는 내장 엔진 + 병렬화로 추가 가속
slopes = df.apply(ols_slope_raw, axis=1, raw=True,
                  engine="numba", engine_kwargs={"parallel": True})

# 나쁨: concat 반복 → O(n^2)에 가까운 비용
# 좋음: 리스트에 모았다가 한 번에 생성
results = pd.Series([fast_slope(r) for r in df.values])
```

---

## 4. String Ops & Effective Pandas (문자열 연산과 효과적 개발)

> 같은 작업도 여러 방법이 있고 오버헤드가 다르며, str 체이닝보다 apply가 중간 Series를 만들지 않아 약 3배 빠를 수 있다.

- `str.split(expand=True)` 체이닝은 여러 중간 Series를 만들어 146ms, 같은 일을 하는 `apply(find_9)`는 51.7ms다.
- numexpr·bottleneck 선택적 의존성을 설치하면 일부 상황에서 추가 속도 향상을 얻는다(설치 안 되어도 경고 없음).
- 계산 후 필터링보다 계산 전 필터링이 낫고, SQL 소스라면 SQL 단계에서 먼저 거른다.
- 저카디널리티 문자열 Series는 `astype('category')`로 바꾸면 value_counts·groupby가 빨라지고 RAM이 준다.
- 메서드 체이닝을 너무 길게 쓰지 말고, 단위 테스트를 추가하며, inplace=True는 제거 예정이므로 피한다.
- 큰 DataFrame은 한 번 전처리 후 to_pickle로 디스크에 저장해 재처리를 피한다.

---

## 5. Dask & Polars (분산 처리와 빠른 DataFrame)

> Dask는 단일 노트북부터 클러스터까지 확장되는 병렬화 프레임워크로 Pandas API를 감싸 larger-than-RAM·멀티코어를 지원하며, Polars는 Arrow+쿼리 최적화기로 더 빠르다.

- Dask 컬렉션: bag(비정형), array(분산 NumPy), distributed dataframe(분산 Pandas), delayed(지연 함수 체인), future(즉시 실행).
- 분산 DataFrame은 인덱스로 행 분할되며 `.compute()`로 지연 계산하고, apply는 axis=1만 지원한다(데이터가 행 청크로 나뉘므로).
- Dask 진단 대시보드(worker별 바이트, task stream, 진행도)로 CPU 바운드/RAM 바운드를 한눈에 파악하며, 트레이스백이 모두 Python이라 PySpark보다 다루기 쉽다.
- 파티션은 코어 수 이상으로 잡고, GIL을 피하려 scheduler="processes"를 쓰면 CPU 바운드 함수에서 이득을 본다.
- 2024년 추가된 dask-expr는 쿼리 최적화로 필요한 행·컬럼만 투명하게 읽어 개발자 개입을 줄인다.
- Swifter는 Dask 위에서 한 줄로 apply를 병렬화하며, 벡터화를 시도하고 안 되면 멀티코어로 실행한다.
- Polars는 더 깨끗한 API, Arrow 단일 백엔드, 내장 쿼리 최적화기, greedy/지연 계산을 지원해 동일 코드에서 보통 2~10배 빠르다(larger-than-RAM은 실험적 streaming 모드).

### Dask 병렬 apply (원문 의도 유지, 새 예제)

```python
import dask.dataframe as dd

ddf = dd.from_pandas(df, npartitions=8, sort=False)   # 코어 수 이상 파티션
results = ddf.apply(
    ols_slope_raw, axis=1, meta=(None, "float64"), raw=True
).compute(scheduler="processes")   # GIL 회피

# Swifter: 한 줄로 병렬화 시도
import swifter
results = df.swifter.apply(ols_slope_raw, axis=1, raw=True)
```

---

## Summary (핵심 정리)

- Pandas에서 가장 빠른 행 단위 연산은 iloc/iterrows 루프가 아니라 apply(raw=True)이며, scikit-learn 같은 안전망 라이브러리는 검증 비용 때문에 느릴 수 있으나 단위 테스트로 보완하면 신뢰성과 속도를 함께 얻는다.
- Numba JIT 컴파일과 병렬화를 결합하면 최대 약 23배 가속이 가능하고, concat 반복 대신 중간 결과를 리스트에 모아 한 번에 구축하며, 문자열은 apply나 category 변환으로 최적화한다.
- larger-than-RAM·멀티코어가 필요하면 Dask(거의 그대로의 Pandas 코드로 확장)와 Swifter를 쓰고, Arrow 기반에 쿼리 최적화기를 내장한 Polars는 더 빠르고 일관된 API를 제공하지만, 모든 벤치마크는 자신의 데이터로 직접 검증해야 한다.
