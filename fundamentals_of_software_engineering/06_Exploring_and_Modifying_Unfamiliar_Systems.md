# 06. Exploring and Modifying Unfamiliar Systems

## 챕터 개요 (3줄 요약)
- 대부분의 개발 업무는 기존 시스템에서 이루어지므로, 낯선 코드베이스를 탐색하고 안전하게 수정하는 기술이 필수다.
- 큰 그림 이해→실행 흐름 추적→점진적 멘탈 모델 구축의 순서로 낯선 코드베이스를 파악한다.
- 테스트에 의존한 안전한 리팩터링, 스카우트 규칙, 작고 되돌릴 수 있는 변경으로 시스템을 안전하게 수정한다.

---

## 1. Understanding Unfamiliar Codebases
> 명확한 계획과 기법으로 접근하면 낯선 코드베이스 작업의 성공률과 자신감을 높일 수 있다.

- 낯선 코드베이스 작업은 스트레스를 주지만, 처음부터 모든 것을 이해하지 못해도 괜찮다.
- 수년간 일한 팀원도 프로젝트의 모든 것을 알지는 못한다.

### Start with the Big Picture
- 코드 이전에 프로젝트의 목적과 의도, 이해관계자, 비즈니스 중요도를 먼저 이해한다.
- 제품 관리자(PM)와 미팅을 잡고 메모/녹화하며 질문한다 — "묻지 않은 질문만이 바보 같은 질문"이다.
- 기술 리드나 아키텍트에게 연락해 의사결정 배경을 듣고, 고객 페르소나로 맥락을 파악한다.
- ADR(Architecture Decision Records)을 포함한 모든 문서를 검토하되, 문서는 거짓말할 수 있으나 코드는 거짓말하지 않는다.
- 온보딩 문서로 로컬 환경을 구성하며 문서의 공백을 메모해 나중에 기여한다.

### Understanding architecture and project structure
- 대부분의 프로젝트는 의도적 설계나 유기적 진화로 어떤 아키텍처 패턴을 따른다.
- Package by layer: controllers, services, repositories, models처럼 기술 책임별 수평 구성.
- Package by feature: users, products처럼 비즈니스 기능별 수직 구성.
- Hexagonal architecture(ports and adapters): 비즈니스 로직을 외부 관심사와 분리.
- Microservices: 기능을 독립적인 다중 서비스로 분리.

```
Package by LAYER          Package by FEATURE
src/                      src/
 controllers/              users/
 services/                  UserController.java
 repositories/              UserService.java
 models/                    UserRepository.java
```

---

## 2. Understand the Execution Flow
> 실행 흐름은 런타임에 프로그램이 따르는 명령의 순차적 경로이며, 진입점을 찾아 추적하면 코드를 이해할 수 있다.

### Finding application entry points
- 애플리케이션은 main 메서드, public API, 웹 UI 등 진입점(door) 집합을 가진다.
- Spring Boot에서는 @SpringBootApplication 어노테이션으로 main 클래스를 찾는다.
- 일반 진입점: main/bootstrap, public API/controller, 이벤트 핸들러, 스케줄 작업, 라이프사이클 훅, 메시지 컨슈머 등.
- IDE 디버거의 브레이크포인트와 단계 실행으로 프레임워크 내부 동작을 파악한다.

### Following the data: Tracing request journeys
- 요청 추적은 컨트롤러, 서비스, 리포지토리 등 관여 계층과 데이터 변환, 숨은 비즈니스 로직을 드러낸다.
- 브라우저 개발자 도구(Network 탭)로 요청/응답 헤더와 페이로드를 검사한다.
- API 테스트 도구(Postman, Insomnia, Bruno)로 UI 없이 엔드포인트를 직접 테스트한다.
- 로깅(logging)은 프로덕션에서 요청 흐름을 추적하는 유일한 도구일 수 있다.
- 디버깅은 코드가 어떤 경로를 "왜" 택하는지 드러내, 동작이 기대와 다를 때 특히 유용하다.

### Locating external dependencies & internal frameworks
- 설정 파일(application.properties/yaml), Docker Compose, Kubernetes 매니페스트에서 외부 의존성(DB, API, 캐시)을 찾는다.
- 사내 자체 프레임워크는 회사별 패키지 접두사, 커스텀 베이스 클래스, 비표준 어노테이션으로 식별한다.
- 사용 예시와 테스트를 먼저 찾고, git blame으로 진화 이력을 파악하며, 배우면서 문서화한다.

---

## 3. Build Mental Models Incrementally
> 멘탈 모델은 시스템 동작에 대한 내부 표현이며, 복잡한 시스템을 작은 조각으로 분해하면 구축이 쉬워진다.

- 전체 애플리케이션이 아닌 체크아웃 같은 단일 경로부터 멘탈 모델을 구축한다.
- 신규 고객 흐름, 체크아웃 흐름, 제품 리뷰 흐름처럼 복잡한 시스템을 관리 가능한 조각으로 나눈다.
- 시각화는 손그림부터 UML까지 가능하며, 플로차트·ER 다이어그램·시퀀스·컴포넌트·마인드맵을 활용한다.
- Mermaid 같은 Markdown 기반 도구로 시작해 점진적으로 세부사항을 추가하며, 생성형 AI로 초안을 만들 수 있다.
- 단일 메서드→클래스와 의존성→기능 수준 흐름→횡단 관심사(보안, 트랜잭션) 순으로 점진적으로 확장한다.

---

## 4. A Sample Process
> Christopher Judd가 가르치는 기존 코드 작업 프로세스로 반복적 이해를 체계화할 수 있다.

- 프로젝트를 SCM에서 클론하고 README, 코딩 표준, 아키텍처 문서를 검토한다.
- 빌드 스크립트, 의존성, 프로젝트 구조, CI/CD 파이프라인을 검토하고 의존성을 설치한다.
- IDE에서 앱과 단위 테스트를 실행/디버그하며 관심 메서드에 브레이크포인트를 건다.
- 커맨드라인에서 아티팩트 빌드, 단위 테스트 실행, 컨테이너 기동, 로컬 실행을 수행한다.
- 첫 패스에 모든 것을 이해할 수 없으며, 이슈/기능 작업마다 더 깊은 지식을 얻는 반복 과정이다.

---

## 5. Making Changes Safely
> 기존 시스템은 메서드적으로 신중히 탐색하며 깨뜨리지 않도록 주의해야 한다("move fast and break things"는 적절치 않다).

### Refactoring Safely
- 포괄적 테스트가 있으면 안전망으로 활용해 작은 변경 후 테스트를 실행하며 부작용을 확인한다.
- 리팩터링 시 동작 보존이 목표이므로 테스트는 수정하지 않는다.
- 테스트 커버리지가 부족하면 테스트 주도 리팩터링(test-driven refactoring)으로 현재 동작을 먼저 문서화한다: 테스트 작성→통과 확인→리팩터링→재확인.
- 버그를 알더라도 먼저 현재 동작을 검증하는 테스트를 작성한 뒤 수정한다.

### The Scout Rule
- 스카우트 규칙: "항상 코드를 발견했을 때보다 더 낫게 남겨라."
- 누락된 문서 추가, 이름 개선, 복잡한 메서드 분해, 죽은 코드 제거, 사소한 버그 수정 등 작은 개선.
- 이해하기 어렵거나, 기능 추가/버그 수정/성능 이슈가 있거나, 여러 개발자가 자주 작업하는 코드는 리팩터링한다.
- 잘 작동하고 변경이 드물거나, 마감 압박, 위험이 이익을 초과하거나, 테스트 커버리지가 부족하면 미룬다.

### Small, Reversible Changes
- 작고 되돌리기 쉬운 변경이 전체 시스템에 영향을 주지 않는 가장 안전한 수정 방법이다.
- 변경 관리 전략: 변경을 가시화, 검증 내장, 롤백 계획(Murphy's law), 효과 모니터링.
- 작은 증분은 위험을 줄이고 테스트·코드 리뷰·문제 해결을 쉽게 한다 — 큰 변경 하나보다 작은 성공 여러 개가 낫다.

### Version control best practices
- 원자적 커밋(atomic commit): 각 커밋은 독립적인 단일 논리 변경을 나타낸다.
- 의미 있는 커밋 메시지로 "무엇을"과 "왜"를 설명하고 자주 커밋한다.
- 기능 브랜치(feature branch)를 만들어 격리하고 짧게 유지하며, 공유 브랜치에서는 rebase를 신중히 사용한다.
- PR(Pull Request)은 단일 관심사에 집중해 작게 유지하고, 코드 리뷰는 코더가 아닌 솔루션 개선에 초점을 둔다.

---

## Summary (핵심 정리)
- 기존 코드 작업은 코더에서 소프트웨어 엔지니어로 성장시키는 핵심 기술이며, 경력 대부분이 기존 코드와 관련된다.
- 큰 그림 이해, 문서/아키텍처 검토, 진입점 찾기와 요청 추적, 점진적 멘탈 모델 구축으로 낯선 코드를 파악한다.
- 테스트 기반 안전한 리팩터링, 스카우트 규칙, 작고 되돌릴 수 있는 변경과 버전 관리 모범 사례로 안전하게 수정한다.