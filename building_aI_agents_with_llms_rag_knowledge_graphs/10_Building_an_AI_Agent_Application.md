# 10. Building an AI Agent Application

## 챕터 개요 (3줄 요약)

- Streamlit으로 에이전트 애플리케이션의 프론트엔드·백엔드를 빠르게 프로토타이핑하는 법을 다룬다.
- MLOps와 LLMOps로 모델을 제품화하고 확장성·복잡성 문제를 해결하는 방법을 설명한다.
- 비동기 프로그래밍(asyncio)과 Docker·Kubernetes 컨테이너화로 시스템을 확장하는 법을 살펴본다.

---

## 1. Introduction to Streamlit

> Streamlit은 Python만으로 백엔드·프론트엔드를 모두 다루는 웹 앱 프레임워크로 빠른 프로토타이핑을 지원한다.

- pip install streamlit으로 설치하고 streamlit run my_app.py로 실행한다.
- Streamlit은 스크립트를 위에서 아래로 순차 실행하며 변경 시 재실행을 안내한다.
- st.title, st.write(스위스 군용칼), st.pyplot 같은 내장 컴포넌트로 텍스트·그래프·표를 표시한다.
- st.map으로 좌표만 주면 대화형 지도를 추가할 수 있다.
- 캐싱: st.cache_data(데이터 결과)와 st.cache_resource(모델·DB 연결 등 자원)로 중복 계산을 방지한다.
- st.session_state로 세션 동안 데이터를 유지하나 페이지 리로드 시 초기화된다.
- st.connection으로 외부 자원에 지속적으로 접근한다.

---

## 2. Developing our frontend with Streamlit

> 텍스트·이미지·대화형 요소로 사용자 경험을 개선해 동적이고 반응적인 앱을 만든다.

- st.title, st.header, st.subheader로 텍스트 구조를 만들며 divider·help 속성을 추가할 수 있다.
- st.set_page_config, st.image, st.caption, st.sidebar.image로 로고·이미지·아이콘을 삽입한다.
- 동적 앱: 슬라이더(st.slider), 체크박스, 두 컬럼 레이아웃으로 인터랙티브 요소를 만든다.
- st.table로 표를, st.progress로 완료율 진행 바를 표시한다.
- FPDF로 PDF를 생성하고 st.download_button으로 다운로드를 제공한다.
- 여러 페이지(노트, 슈퍼마켓 찾기)를 사이드바로 추가하며 OSM Overpass API로 지도를 그린다.

---

## 3. Creating an application with Streamlit and AI agents

> 9장의 다중 에이전트 여행 계획 시스템을 Streamlit 앱에 캡슐화해 비전문가도 사용할 수 있게 한다.

- 여행 계획 시스템은 WeatherAnalysisAgent, HotelRecommenderAgent, ItineraryPlannerAgent, SummaryAgent 네 에이전트로 구성된다.
- 핵심 모델: RandomForestRegressor(날씨), SentenceTransformer(호텔), GPT-2/GPT-4(일정·요약).
- UML 다이어그램(클래스·활동·시퀀스)으로 아키텍처와 실행 흐름을 시각화한다.
- ItineraryPlannerAgent는 OpenAI GPT-4로 더 매끄러운 일정을 생성한다.
- st.secrets로 API 키를 안전하게 관리해 하드코딩 위험을 줄인다.
- 버튼 클릭 시 최적 월 예측, 호텔 추천, 일정 생성, 지도 표시가 실행된다.

---

## 4. Machine learning operations and LLM operations

> MLOps는 ML 모델 수명주기를 관리하며, LLMOps는 대규모 NLP 모델의 추가 복잡성을 다룬다.

- MLOps 단계: 모델 개발, 테스트, 배포, 모니터링·유지보수.
- LLMOps는 모델 크기, 학습·파인튜닝, 확장성, 모니터링, 거버넌스에서 추가 복잡성을 가진다.
- LLM 모니터링은 정확도뿐 아니라 텍스트 품질, 편향·유해 출력 등을 평가해 인간 개입(human-in-the-loop)이 필요하다.
- 모델 개발은 일반 데이터(웹·책·대화)와 특화 데이터(다국어·과학·코드) 코퍼스 수집으로 시작한다.
- 전처리: 저품질 제거, 중복 제거(deduplication), PII 제거, BPE 토큰화.
- 학습은 데이터 혼합(data mixture)과 데이터 커리큘럼(data curriculum) 비율을 정한다.
- ETL(extract, transform, load) 파이프라인과 RBAC(role-based access control), 피처 스토어를 사용한다.

---

## 5. Asynchronous programming

> 비동기 프로그래밍은 메인 스레드를 막지 않고 작업을 동시에 처리해 I/O 바운드 작업의 성능을 높인다.

- I/O 바운드는 외부 시스템 대기, CPU 바운드는 집약적 연산이 주 제약이다.
- 동시성(concurrency)은 작업을 겹치는 시간에, 병렬성(parallelism)은 정확히 동시에 처리한다.
- 핵심 개념: 블로킹/논블로킹, 콜백, 프로미스/퓨처, 이벤트 루프(event loop), 코루틴(coroutine).
- asyncio는 async/await 구문으로 멀티스레딩 없이 동시 코드를 작성한다.
- asyncio.gather()는 모두 대기, asyncio.create_task()는 유연한 백그라운드 실행을 제공한다.
- ML 적용: 데이터 로딩, 하이퍼파라미터 튜닝, 비동기 추론, 모델 학습에 활용한다.
- LLM 에이전트의 비동기 함수 호출은 토큰 생성과 함수 실행을 겹쳐 지연을 줄인다.

### LLM 에이전트 함수 호출 세 방식

```
synchronous          : functions run one by one (high latency)
sync + parallel opt  : concurrent but still blocking
asynchronous         : non-blocking, dynamic dependencies (best latency)
```

---

## 6. Docker

> Docker는 애플리케이션과 의존성을 컨테이너로 패키징해 어느 환경에서나 일관되게 실행한다.

- 컨테이너는 가상머신과 달리 OS 커널만 포함해 가볍고 효율적이다.
- 핵심 개념: 컨테이너, 이미지(읽기 전용 템플릿), Docker Engine, Dockerfile, Docker Compose.
- 장점: 이식성, 효율성, 격리(isolation), 버전 관리·재현성.
- 단점: 보안 우려, 데이터 관리(컨테이너는 휘발성), 복잡성.
- Kubernetes는 컨테이너 오케스트레이션 플랫폼으로 Pod, Service, Node, Cluster로 구성된다.
- Docker는 ML 재현성·배포에 널리 쓰이며 Ollama 이미지로 LLM 앱을 배포할 수 있다.
- Repo2Run, LLMSecConfig처럼 LLM으로 Dockerfile 생성·보안 설정을 자동화할 수 있다.

### Docker vs 가상머신

```
VM        : full guest OS -> heavy
Container : app + deps + kernel only -> lightweight, portable
Kubernetes: orchestrates many containers (Pod/Service/Node/Cluster)
```

---

## Summary (핵심 정리)

- Streamlit으로 백엔드·프론트엔드를 빠르게 프로토타이핑해 다중 에이전트 시스템을 앱으로 만드는 법을 배웠다.
- MLOps·LLMOps로 모델을 제품화하고 비동기 프로그래밍으로 성능을 높이는 법을 익혔다.
- Docker·Kubernetes로 시스템을 격리·확장했으며, 다음 장에서 에이전트 분야의 미래 전망을 다룬다.
