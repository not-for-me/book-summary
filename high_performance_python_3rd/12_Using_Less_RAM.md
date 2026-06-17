# 12. Using Less RAM

## 챕터 개요 (3줄 요약)

- "데이터는 질량을 가진다" — RAM을 적게 쓰면 데이터가 버스·캐시를 더 빨리 이동하고 한 머신에 다 담겨 클러스터를 피할 수 있다.
- 숫자는 array/numpy로, 문자열은 trie(접두사 트리)로 압축 저장하며(1.2GB→39MB), NumExpr로 중간 배열을 줄여 RAM과 시간을 함께 절약한다.
- 정확도를 저장공간과 맞바꾸는 확률적 자료구조(Morris counter, KMV, Bloom filter, HyperLogLog)는 극적으로 적은 RAM으로 근사 카운팅·집합 멤버십을 답한다.

---

## 1. Primitives & array/numpy (원시 객체와 배열)

> 각 고유 Python 객체는 메모리 비용을 가지므로, 1억 개의 서로 다른 정수를 list에 담으면 수 GB가 들지만 array/numpy는 연속 메모리에 저렴하게 저장한다.

- 같은 객체 참조 1억 개 list는 약 760MB지만, 서로 다른 정수 1억 개는 약 3.1~3.8GB가 든다(각 고유 int가 28바이트+).
- `array` 모듈은 정수·실수·문자를 연속 블록에 저장해 1억 정수를 약 760MB로 담으며(타입코드로 정밀도 선택), 복소수·클래스는 불가.
- numpy 배열은 더 넓은 타입(complex128=16바이트), itemsize·nbytes 조회를 지원하며, zeros는 지연 할당이라 RAM을 늦게 쓴다.
- array/numpy 값은 Python 객체가 아니라 바이트라, 역참조하면 새 Python 객체가 생겨 계산 시 이득이 사라지지만, 외부 프로세스 전달·일부 사용엔 큰 절약이다.
- 수치 작업엔 numpy가, 의존성 최소화엔 array가 적합하며 Cython은 둘 다, Numba는 numpy만 지원한다.

### array vs numpy 메모리 (원문 의도 유지, 새 예제)

```python
import array, numpy as np

# list: 고유 정수 1억 개 → 수 GB
# array: 연속 메모리에 저렴하게 (타입코드 'l' = signed long)
arr = array.array('l', range(100_000_000))   # 약 760MB

# numpy: 타입·메모리 조회 가능
a = np.ones(100_000_000, np.complex128)
print(a.nbytes, a.itemsize)   # 1.6GB, 16바이트/항목
```

---

## 2. NumExpr & Measuring RAM (NumExpr와 RAM 측정)

> 큰 NumPy/Pandas 벡터 연산은 보이지 않는 큰 중간 배열을 만들어 RAM을 소모하고 캐시 비친화적이지만, NumExpr은 캐시 친화적 청크로 나눠 병렬 처리해 추가 RAM 없이 빠르다.

- 교차 엔트로피 계산에서 직접 벡터 연산은 여러 임시 배열로 현재 위에 4.5GB가 더 필요(총 9GB+)하고 10초 걸리지만, numexpr.evaluate는 추가 RAM 0에 0.6초다.
- NumExpr은 긴 벡터를 짧은 캐시 친화적 청크로 나눠 병렬 처리하므로 RAM과 속도를 모두 개선한다.
- Pandas의 df.eval은 NumExpr이 설치되어 있으면 빨라지지만(미설치 시 경고 없이 느림), 반드시 함께 설치를 권장한다.
- `sys.getsizeof`는 컨테이너 내용물 비용을 세지 않으므로(list는 항목당 8바이트만), pympler의 asizeof가 더 낫지만 느리고, memit이 실제 프로세스 RAM을 측정해 가장 유용하다.
- Python 3.x는 기본 Unicode이며 PEP 393 덕분에 ASCII 문자는 1바이트, 흔치 않은 문자(Σ 등)는 더 많은 바이트를 쓴다.

### NumExpr로 중간 배열 제거 (원문 의도 유지, 새 예제)

```python
import numpy as np, numexpr

yp = np.random.uniform(low=1e-7, size=200_000_000)
yt = np.ones(200_000_000)

# 직접: 여러 임시 배열 → RAM 폭증, 느림
ans = -(yt * np.log(yp) + (1-yt) * np.log(1-yp))

# NumExpr: 캐시 청크 + 병렬 → 추가 RAM 0, 빠름
ans = numexpr.evaluate("-(yt * log(yp) + (1-yt) * log(1-yp))")
```

---

## 3. Efficient Text with Tries (trie로 텍스트 효율 저장)

> 문자열을 순진하게 저장하면 비싸지만, trie(접두사 트리)와 DAWG는 공통 접두사·접미사를 공유해 1.2GB 문자열 집합을 39MB로 압축하면서 빠른 조회를 유지한다.

- 1,250만 고유 토큰 저장 비교: list(느림·830MB), bisect 정렬 list(0.06초·883MB), set(0.01초·1.3GB), dict(1.6GB).
- trie는 공통 접두사만, DAWG는 접두사+접미사를 공유해 RAM을 절약하며, 공통 접두사 검색(prefix search)이 빠르다.
- Marisa trie(정적, Cython 바인딩)는 빌드 후 디스크에 저장하고 재로딩이 매우 빠르며 RAM 효율적(로딩 후 약 90MB, 조회 0.06초).
- 실제 사례: DabApps는 영국 170만 우편번호를 dict(수백 MB) 대신 Marisa trie로 30MB에 담아 무료 호스팅에 배포했다.
- trie는 덜 알려졌지만 겹치는 시퀀스(DNA 등)에서 큰 RAM 이득을 주며, 별도 모듈로 격리해 유지보수를 단순화하길 권장한다.

---

## 4. FeatureHasher & Sparse Matrices (특징 해싱과 희소 행렬)

> scikit-learn의 FeatureHasher는 어휘(vocabulary)를 저장하지 않고 해싱으로 고정 너비 희소 행렬을 만들어, DictVectorizer보다 빠르고 RAM을 절약하면서 동등한 머신러닝 성능을 낸다.

- NLP에서 n-gram 어휘는 폭발하므로, stop-word 제거·소문자화·희귀어 무시로 통제한다.
- DictVectorizer는 어휘를 두 번 패스로 빌드해 가변 너비 희소 행렬을 만들고 역변환이 가능하지만 느리고 RAM을 더 쓴다.
- FeatureHasher는 MurmurHash3로 토큰을 고정 너비(기본 100만 열)에 매핑해 한 번 패스로 빠르며, 충돌이 있어도 신호가 충분하다(역변환 불가).
- 20 Newsgroups에서 DictVectorizer(400만 열, 36초 빌드)와 FeatureHasher(100만 열, 14초)가 같은 0.89 점수를 내고, 후자가 학습도 70% 빠르다.
- SciPy 희소 행렬(sparse matrix)은 0이 대부분인 행렬을 비0 값만 저장해, 저밀도에서 dense보다 메모리·속도가 훨씬 낫다(코사인 유사도에 특히 유용).

### FeatureHasher (원문 의도 유지, 새 예제)

```python
from sklearn.feature_extraction import FeatureHasher

# 어휘 저장 없이 고정 100만 열로 해싱 → 빠르고 RAM 절약
fh = FeatureHasher(n_features=1_000_000, alternate_sign=False)
token_dicts = [{'there': 1, 'is': 1, 'a': 1, 'cat': 1}]
X = fh.transform(token_dicts)   # 희소 행렬, 역변환 불가
```

---

## 5. Probabilistic Data Structures (확률적 자료구조)

> 정확도를 RAM과 맞바꾸는 확률적 자료구조는 손실 압축(lossy compression)처럼 특정 질문에 필요한 정보만 유지해, 극적으로 적은 메모리로 근사 답을 준다.

- Morris counter는 지수만 저장해 2^exponent로 근사 카운팅하며, 1바이트로 약 5e76까지 셀 수 있지만 중복 제거는 안 한다.
- KMV(K-Minimum Values)는 해시의 균등 분포를 이용해 가장 작은 k개 고유 해시로 고유 항목 수를 추정하며 멱등성(idempotence)을 가진다(union/intersection 지원).
- Bloom filter는 여러 해시 인덱스를 bit 배열에 표시해 "본 적 있나?"에 답하며, false negative는 없고 제어 가능한 false positive율을 가진다(scalable·timing 변형 존재).
- HyperLogLog는 해시 비트의 무작위성(앞쪽 0의 개수)으로 카디널리티를 추정하며, 2.56KB로 약 79억 항목을 1.625% 오차로 센다(register 추가로 정확도 향상).
- 자료구조 선택은 먼저 데이터에 묻고 싶은 질문(union/intersection/contains)을 이해한 뒤, 오차율과 메모리 트레이드오프로 결정한다.
- Wikipedia 고유 단어 약 495만 개: HyperLogLog는 40KB에 -0.54% 오차, 실제 정확 저장은 49MB로, 확률적 구조가 코드 특화로 큰 메모리·속도 이득을 준다.

### Bloom filter 개념 (원문 의도 유지, 새 예제)

```python
import math, bitarray, mmh3

class BloomFilter:
    def __init__(self, capacity, error=0.005):
        self.num_bits = int(-capacity * math.log(error) / math.log(2)**2 + 1)
        self.num_hashes = int(self.num_bits * math.log(2) / capacity + 1)
        self.data = bitarray.bitarray(self.num_bits)
        self.data.setall(False)

    def _indexes(self, key):
        h1, h2 = mmh3.hash64(key)        # double hashing
        for i in range(self.num_hashes):
            yield (h1 + i * h2) % self.num_bits

    def add(self, key):
        for idx in self._indexes(key):
            self.data[idx] = True

    def __contains__(self, key):         # false negative 없음
        return all(self.data[idx] for idx in self._indexes(key))
```

---

## Summary (핵심 정리)

- RAM을 적게 쓰면 데이터가 캐시·버스를 빨리 이동해 더 빠르고 한 머신에 담기므로, 숫자는 array/numpy(희소 데이터는 SciPy sparse)로, 문자열은 str+trie로 저장하고 NumExpr로 중간 배열을 줄인다.
- 실제 RAM 측정은 sys.getsizeof보다 memit(실제 프로세스 측정)이 신뢰할 만하며, NLP에서는 어휘를 저장하지 않는 FeatureHasher가 DictVectorizer보다 빠르고 RAM 효율적이다.
- 정확도를 저장공간과 맞바꾸는 확률적 자료구조(Morris/KMV/Bloom/HyperLogLog)는 코드 특화의 한 형태로, 데이터에 묻고 싶은 질문을 먼저 정한 뒤 오차율·메모리 트레이드오프에 맞는 구조를 골라 극적인 RAM 절약을 얻는다.
