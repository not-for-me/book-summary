# 13. Testing AI Systems

## 챕터 개요 (3줄 요약)
- AI system은 코드+데이터를 모두 테스트해야 하므로 전통 소프트웨어보다 testing pyramid가 높다(offline + online).
- dev→staging→prod 여정을 version control·CI/CD·automatic containerization으로 지원하고, FTI pipeline마다 다른 integration test를 작성한다.
- model deployment는 blue/green·A/B로 검증하고, agent는 eval로 평가하며, governance(schematized tag·lineage·versioning·audit log)로 통제한다.

---

## 1. Offline Testing & Dev to Prod
> offline test + online check를 lifecycle 전반에 적용. testing pyramid(아래=많은 unit/validation, 위=적은 deployment test).

- AI는 작은 버그가 silent 오예측 유발 → 코드+데이터 테스트. CI/CD는 전제조건 아님(MVPS 후 점진 추가).
- 인프라: version control, CI/CD(GitHub Actions/Jenkins/GitLab), artifact repo(PyPI/Maven), container registry, feature store/model registry, serving, agent 배포.
- Hopsworks: 환경별 project(prod는 별도 클러스터). artifact는 기본 미이동(비prod 데이터로 생성), **코드만 PR로 이동**(테스트 트리거, env var로 dev/staging/prod 구분, DRY). 인간 리뷰 후 prod.
- pre-commit hook(black/flake8/bandit/nbstripout).

---

## 2. Automatic Containerization & Jobs
> Dockerfile 작성 대신 자동 컨테이너화로 프로그램·job에만 집중.

- 직접 컨테이너화(Dockerfile→compile→registry→orchestrator) vs **automatic containerization**(소스에서 자동 빌드·등록·실행).
- base image(OS 패키지) + base Python env(라이브러리)에서 빌드.
- **Hopsworks**: FTI별 base container 선택·커스터마이즈(UI/코드). env 생성 + PySpark job(driver/executor 리소스, cron). Papermill로 notebook 실행. Airflow로 job DAG.
- **Modal**: 프로그램 수준 자동 컨테이너화(`debian_slim().apt_install().pip_install()` 데코레이터). 외부 실행이라 env var(API key) 주입 필요.

---

## 3. CI/CD Tests for AI Systems
> FTI pipeline마다 다른 integration test. feature(데이터·invariant), training(bias·성능), inference(품질·SLO).

### Feature Pipeline Tests
- 함수로 refactor(데이터 소스·validation·pipeline·main). sample 데이터(synthetic, compliance 회피) commit. dev feature store에 쓰기.
- test: 대상 feature group drop/재생성, sample insert(expectation suite ALWAYS), validation_report·job 대기, 행 수 assert. 외부 연결 mock하면 unit test(빠르나 e2e 아님).

### Training Pipeline Tests
- 출력=trained model, 시간 소요, 학습 후 항상 model validation. 성능 범위·bias 검사(예: online/offline merchant별 f1, feature view filter로 evaluation data).
- dev 성공 → prod 전체 학습(수동/CI) → model deployment owner 승인(blue/green).

### Model Deployment Tests
- **A/B test**: 트래픽 X%/Y% 분할, 애플리케이션 KPI(CTR/engagement/conversion) 비교(모델의 앱 영향 테스트).
- **blue/green**: 100% prod(blue) + Y% challenger(green), client 무위험. 로그 비교 후 교체.
- batch inference A/B: live 대신 historical backtesting, 단일 scalar 성능으로 best model 선택(`get_best_model`).

### Evals for Agents
> agent는 다단계라 step별 테스트 대신 curated prompt에 대한 응답을 e2e 평가.

- LLM·system prompt·RAG query·RAG 데이터 변경이 응답 품질에 영향.
- **evals**: prompt + expected response 데이터셋(eval_id/task/prompt/expected_response). feature store에 저장. staging agent 실행 → traces → eval runs.
- **evaluator**: traces·expected를 score(subjective는 LLM-as-judge, objective는 task별 프로그램). 검사: hallucination, safety(toxicity/bias/PII/injection). 점수: binary 또는 Likert 1-5 + feedback.
- backfill과 유사(historical 입력). RAG eval(chunk attribution/utilization/relevance).
- **synthetic eval 생성**: 다양성, user input 생성(output 아님), 시스템 제약 반영, 시나리오 커버리지, frontier LLM.
- **point-in-time correct RAG**: mutable 소스 재실행 시 다른 결과 → time travel + timestamp 필요(현재 vector index/online store 미지원). 해법: immutable 합성 소스 또는 production request를 TTL eval로 로깅.

---

## 4. Governance
> 규제·정책 준수 통제. schematized tag·lineage·versioning·audit log.

- AI data governance: bias 없는 training data, 결정 traceability, 정확·robust·secure, 인간 oversight. data quality·access control·lineage·auditing.

### Schematized Tags
- custom metadata로 AI asset 기술·discovery·정책 정의. tag(keyword, 자유형) vs **schematized tag**(스키마 준수: name/type/required/valid 범위).
- 예: EU AI Act 태그(risk_level enum, conformity_date 등). JSON 정의, asset에 attach, free-text search. 특정 태그 없으면 prod 생성 차단 가능.
- 예: GDPR(retention date), compliance(지역/PII), checklist(owner/consumer/harm).

### Lineage
> 데이터·모델의 origin·변환·이동을 추적.

- 그래프: Data Source → Feature Group → Feature View → Training Data → Model → Deployment(양방향 traversal).
- graph API로 provenance query(이 feature group을 쓰는 모델? 이 feature view의 feature group?). tag+provenance로 governance enforcement job.

### Versioning
- feature group(mutable, data versioning, schema-breaking 시 새 버전+backfill), feature view(immutable, 저렴), training data(immutable, 비쌈), model(immutable, 학습마다), deployment(mutable, blue/green, deployment API 의존).
- schema-breaking(계산 방식 변경/삭제/타입 변경)은 새 버전. Iceberg branch/tag로 PB급 데이터 실험.

### Audit Logs
- tamperproof 기록: feature store event, model lifecycle, access control, deployment activity. model validation report·model card·dashboard.

---

## Summary (핵심 정리)
- MLOps의 offline testing을 다뤘다(version control, CI/CD, dev/staging/prod 인프라).
- FTI pipeline의 다양한 offline test를 작성했다.
- blue/green test로 model deployment를 prod 전 평가한다.
- governance rule 설계·enforcement와 lineage·versioning으로 안전한 디버깅·업그레이드를 했다.
- eval로 LLM inference 변경 성능을 평가한다.
