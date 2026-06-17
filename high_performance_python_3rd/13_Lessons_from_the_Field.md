# 13. Lessons from the Field

## 챕터 개요 (3줄 요약)

- 여러 회사의 핵심 인물들이 고데이터·고속 환경에서 Python을 쓴 실전 경험과 어렵게 얻은 지혜를 공유하는 사례 모음 장이다.
- 공통 교훈은 "올바른 문제를 풀어라" — 데이터를 직접 들여다보고, 요구사항을 첫 원리부터 이해하며, 종종 머신러닝보다 단순한 SQL/DB 쿼리가 더 빠른 해법이다.
- 기술 최적화(2~10배)보다 문제 재정의·접근법 변경이 10~100배 이득을 주며, 유연성·도메인 지식·방어적 코딩·인간 워크플로 보강 도구가 실전 고성능의 열쇠다.

---

## 1. ML Algorithm & Journalism (머신러닝 알고리즘과 저널리즘)

> Evolution AI의 Martin Goodson은 가설 기반 연구(hypothesis-driven research)로 18개월간 텍스트 검출 CNN을 개선했고, Bloomberg의 Leon Yin은 빠른 반복으로 인터넷 차별을 폭로했다.

- 데이터를 충분히 들여다보지 않아(eyeballing) 합성 데이터 생성 버그를 일주일 훈련 후에야 발견 — 한두 시간의 사전 검토로 막을 수 있었다.
- 맞춤형 분석 도구(heatmap, 인터랙티브 산점도)를 만들자 연구가 가속되어, 긴 단어 검출 약점을 발견하고 직사각형 커널로 해결했다.
- 학술 객체 검출 지표는 실제 문서 텍스트 검출에 부적합·오해 소지가 있어, 실제 사용 사례에 맞는 새 평가 지표를 만들어야 했다.
- 저널리즘에서 Selenium 스크래퍼는 2주에 4,000주소(IP 차단·렌더링 오버헤드)였지만, 미공개 API + aiohttp 비동기 + session 객체로 하루 30만 주소로 가속(약 1000배).
- 빠르고 반복적이며 실용적으로 시작(사전 연구·전문가 인터뷰·소규모 표본)한 뒤, 잠재력이 확인되면 고성능 엔지니어링을 투입한다.

### 동기 vs 비동기 스크래핑 (원문 의도 유지, 새 예제)

```python
import aiohttp

# session으로 쿠키·헤더를 요청 간 유지하며 미공개 API 체이닝
async def get_plans(address, proxy):
    async with aiohttp.ClientSession() as session:
        await session.get('site/authenticate', proxy=proxy)
        addr_id = await session.get('site/autocomplete',
                                    json={"address": address}, proxy=proxy)
        plans = await session.post('site/plans',
                                   json={'addressId': addr_id}, proxy=proxy)
        return await plans.json()
```

---

## 2. Cyber Reinsurance & Quant Finance (사이버 재보험과 퀀트 금융)

> Gallagher Re의 James Poynter는 요구사항을 첫 원리부터 이해하고 방어적 코딩으로 나쁜 데이터를 막으며, Engelhart의 Mikhail Timonin은 실시간과 백테스팅 성능의 균형을 강조한다.

- 1순위 교훈: 고성능은 요구사항을 첫 원리부터 진짜 이해하는 데서 시작하며, "작성하지 않은 코드"가 종종 더 중요하다(삭제·단순화·간소화).
- 도메인 지식은 과소평가되며, 올바른 문제를 올바른 방식으로 풀게 해준다.
- 방어적 코딩: assert(상태 검증), try-except(우아한 예외 처리), logging(데이터 변환 추적), 타입 힌트, 테스트 프레임워크(PyTest), Pandera(데이터프레임 스키마 검증).
- 기술 부채는 복리로 쌓이므로 Azure ML·MLflow 같은 PaaS·검증된 패키지로 ML 라이프사이클(데이터·훈련·배포·모니터링·재훈련)을 관리한다.
- 완전 자동화가 불가능한 곳엔 Streamlit·Dash 같은 저코드 프레임워크로 인간 중심 워크플로를 보강하는 도구를 만든다.
- 퀀트 금융: 벡터화(Pandas/NumPy)는 백테스팅엔 좋지만 경로 의존성·룩어헤드 편향·메모리 문제가 있고, 경량 상태 머신은 실시간엔 좋지만 효율적 코드 작성이 어렵다.
- 설계 팁: 모듈화(divide and conquer), Numba+벡터화 결합, Pandas보다 Polars, 백테스팅은 Kubernetes로 수평 확장.

### Pandera 데이터 검증 (원문 의도 유지, 새 예제)

```python
import pandera as pa

schema = pa.DataFrameSchema({
    "premium": pa.Column(float, pa.Check.greater_than(0)),
    "risk_score": pa.Column(int, pa.Check.in_range(0, 100)),
    "client_id": pa.Column(str, nullable=False),
})

# 런타임에 데이터 품질 검증 → 실패 시 명확한 오류 메시지
validated_df = schema.validate(raw_df)
```

---

## 3. Flexibility & Solving the Right Problem (유연성과 올바른 문제 풀기)

> WSP의 David Rawlinson은 코드 유연성 유지가 최고의 고성능 조언이며, Vincent Warmerdam은 잘못된 문제를 최적화한 경고성 사례를 들려준다.

- 정의된 프로세스 최적화는 보통 2~10배지만, 새 아이디어·접근법은 10~100배 가속을 가능하게 하므로 유연성을 철저히 유지해야 한다.
- 코드 복잡도는 최적화 능력을 제한하며, 조기 라인별 최적화는 나중의 급진적 리팩토링을 막을 수 있다.
- 문제를 재정의하면 종종 문제 자체가 사라진다(런타임 0) — 빠른 근사 해법(bounding box·K-D Tree 공간 인덱스) 후 느린 정확 해법을 쓰는 트릭.
- 트럭 도착 예측 사례: 18개월간 복잡한 시계열 ML 모델을 최적화했지만, 사실 "카트 대여 테이블"을 보면 단순 SQL 프로젝션으로 풀 수 있는 문제였다(잘못된 문제를 풀고 있었다).
- 좋은 데이터는 ML 프로젝트에 필수이며, 데이터를 만드는 작업 자체가 원래 목적과 무관하게 큰 가치를 가진다.

---

## 4. Numba, Optimization & word2vec (Numba와 최적화)

> Anaconda의 Valentin Haenel은 Numba의 @jit으로 손쉬운 가속을, Radim Řehůřek은 Python word2vec을 Google C 원본보다 4배 빠르게 만든 최적화 여정을 공유한다.

- Numba는 @jit(nopython=True) 데코레이터만으로 수치 코드를 LLVM JIT 컴파일해 약 400배 가속이 가능하며, NumPy를 인식한다.
- 모범 사례: nopython 모드 우선, NumPy 배열·typed-list/typed-dict 사용(이종 Python list/dict는 미지원), for 루프도 빠름(루프 융합·인라이닝).
- word2vec 최적화 교훈: 데이터 스트리밍(제너레이터로 일정 메모리), NumPy 생태계 활용, Cython으로 핫스팟 컴파일, BLAS 활용(axpy로 4배), 멀티스레딩(GIL 해제로 3배), 정적 메모리 할당.
- 인간 수준 최적화(소통·문제 정의·KISS·데이터 수동 점검·유행 신중)가 기계 수준 최적화보다 먼저다.
- 데이터 파이프라인에 간단한 로그를 뿌려 데이터를 사람이 읽을 수 있게 보면 "aha!" 순간이 자주 온다.

### Numba @jit (원문 의도 유지, 새 예제)

```python
import numpy as np
from numba import njit

@njit                          # nopython 모드, 첫 호출 시 컴파일
def sieve(n):
    flags = np.ones(n, dtype=np.uint8)
    for i in range(2, n):
        if flags[i]:
            for x in range(i + i, n, i):   # for 루프도 빠름
                flags[x] = 0
    return np.nonzero(flags)[0][2:]
```

---

## 5. Teams, Feature Engineering & Smesh (팀·특징 공학·소셜 분석)

> Linda Uruchurtu는 고성능 데이터 과학 팀 운영을, Soledad Galli는 Feature-engine 오픈소스를, Alex Kelly는 Aho-Corasick로 실시간 문자열 매칭을 100배 가속한 사례를 공유한다.

- "얼마나 걸릴까?"에 답하기 전에 문제 범위(왜 문제인가, done의 정의, 최소 해법)를 정의하고, 발견·계획 단계로 불확실성을 줄인 MVP를 먼저 만든다.
- scope creep, 비기술 작업 과소평가, 테스트 부족이 일정을 지연시키므로 주간 정제·계획 세션으로 관리한다.
- Feature engineering 파이프라인은 데이터에서 파라미터를 자동 학습·저장하고, 연구·운영 환경에서 같은 코드를 써 재현성을 높여야 한다(Feature-engine은 scikit-learn fit/transform 활용).
- 잘 확립된 오픈소스(문서화·테스트·커뮤니티 검증)를 쓰면 코드 작성을 덜고 팀 성능·재현성·협업을 높인다.
- Smesh: 초당 수백 트윗을 수천 regex로 매칭하는 병목을, Aho-Corasick 트라이를 prefilter로 써 문제 공간을 줄여 10~100배 가속했다.
- 모니터링(Graphite/Sentry)·배포(Puppet/Ansible/Salt)·고가용성(지리적 분산 클러스터+낮은 TTL DNS)으로 시스템을 안정적으로 운영한다.

---

## Summary (핵심 정리)

- 실전 고성능의 가장 큰 교훈은 "올바른 문제를 풀어라"이며, 데이터를 직접 들여다보고 요구사항을 첫 원리부터 이해하면 종종 복잡한 ML보다 단순한 SQL·근사 해법이 정답이다.
- 기술 최적화(2~10배)보다 문제 재정의·접근법 변경이 10~100배 이득을 주므로 코드 유연성을 유지하고, Numba·Cython·BLAS·비동기 I/O 같은 도구는 핫스팟에만 집중 투입한다.
- 방어적 코딩(assert·로깅·Pandera·테스트)으로 나쁜 데이터를 막고, 검증된 오픈소스·PaaS로 기술 부채를 관리하며, 완전 자동화가 불가능한 곳엔 Streamlit 같은 도구로 인간 워크플로를 보강하는 것이 팀 단위 고성능의 핵심이다.
