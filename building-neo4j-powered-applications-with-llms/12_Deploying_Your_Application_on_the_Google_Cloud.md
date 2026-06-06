# 12. Deploying Your Application on the Google Cloud

## 챕터 개요 (3줄 요약)

- Chapter 5의 Haystack 챗봇을 Google Cloud Run에 serverless로 배포하는 실습 워크플로를 다룬다.
- Docker 컨테이너화 → Google Cloud 프로젝트·서비스 설정 → Cloud Run 배포 → 검증 단계를 안내한다.
- 프로덕션 배포를 위한 Neo4j 배포 아키텍처(Primary/Secondary)와 데이터 ingestion·augmentation 고려사항을 제시한다.

---

## 1. Preparing your Haystack chatbot for deployment

> 컨테이너화 전에 코드를 serverless 배포에 맞게 구성하고 필수 설정 파일을 준비한다.

- Chapter 5의 search_chatbot.py를 app.py로 복사·이름 변경(많은 클라우드 서비스의 기본 진입점).
- requirements.txt: haystack-ai==2.5.0, openai, gradio, python-dotenv, neo4j, neo4j-haystack 등 의존성 명시.
- .env 파일(example.env 템플릿 복사)로 OPENAI_API_KEY, NEO4J_URI/USERNAME/PASSWORD 등 민감 정보를 코드 밖에서 관리(python-dotenv로 로드).
- 디렉터리 구조: app.py, requirements.txt, Dockerfile, .env, example.env.
- 로컬 Neo4j 사용 시 ngrok(`ngrok tcp 7687`)으로 bolt 포트를 공개해야 하며, AuraDB Free는 불필요.

---

## 2. Containerizing the application with Docker

> 앱을 Docker 컨테이너로 패키징하여 일관된 실행 환경을 만든다.

```
FROM python:3.11
EXPOSE 8080
WORKDIR /app
COPY . ./
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

- python:3.11 base image, Cloud Run이 기대하는 port 8080 노출.
- WORKDIR /app, 프로젝트 전체 복사, requirements.txt 의존성 설치, app.py 실행.

---

## 3. Setting up a Google Cloud project and services

> Cloud Run, Artifact Registry, Cloud Build로 코드를 serverless 배포까지 연결한다.

- 프로젝트 생성·billing 활성화(console.cloud.google.com).
- Google Cloud Shell 사용(Docker·gcloud CLI·Git 사전 설치, 임시 VM 5GB 저장소).
- 활성 프로젝트 설정: `gcloud config set project YOUR_PROJECT_ID`.
- 필요 서비스 활성화: `gcloud services enable`로 cloudresourcemanager, servicenetworking, run, cloudbuild, cloudfunctions API.
- 프로젝트 파일은 업로드하거나 GitHub(ch12)에서 clone.

---

## 4. Deploying to Google Cloud Run

> 환경변수 설정부터 Artifact Registry 구성, 컨테이너 빌드, Cloud Run 배포까지 진행한다.

- 환경변수 export: GCP_PROJECT, GCP_REGION(예: us-central1), AR_REPO, SERVICE_NAME.
- Artifact Registry 생성: `gcloud artifacts repositories create` (--repository-format=Docker), Docker 인증.
- 빌드·푸시: `gcloud builds submit --tag "$GCP_REGION-docker.pkg.dev/$GCP_PROJECT/$AR_REPO/$SERVICE_NAME"`.
- .env를 --set-env-vars 형식으로 변환 후 `gcloud run deploy` (--port=8080, --allow-unauthenticated, --platform=managed).
- 완료 시 챗봇이 live되는 public URL 반환.

```
[Dockerfile] -> Cloud Build -> Artifact Registry (image)
   -> Cloud Run deploy (--set-env-vars) -> public service URL
```

---

## 5. Testing and verifying / Other clouds

> Cloud Run의 public URL로 Gradio 챗봇 인터페이스를 검증한다.

- URL 형식: `https://movies-chatbot-[UNIQUE_ID].{GCP_REGION}.run.app`.
- 트러블슈팅: 의존성 확인, 인증(service account 권한), 로그·메트릭 모니터링, Cloud Run 서비스 관리(URL·배포 이력·환경변수·로그).
- 다른 클라우드: docker compose 준비 후 Azure(Container Apps)·AWS(ECS) 가이드 참고. Spring Boot 앱(Chapter 9·10)은 Cloud Run·Azure Spring Apps·AWS EC2/Elastic Beanstalk로 배포 가능.

---

## 6. Preparing for deployment in production: key considerations

> 프로덕션은 데이터 ingestion·consumption, LLM/ML 파이프라인, graph DB 확장 아키텍처를 고려해야 한다.

### Neo4j 배포 아키텍처 (Figure 12.1)

- Primary(READ+WRITE): 일반 상호작용·데이터 ingestion·consumption 처리. 고가용성을 위해 다수 가능.
- Secondary(READ only): 분석·GDS 워크로드(graph augmentation) 처리. 수평 확장을 위해 다수 가능.
- Neo4j Ops Manager로 배포·모니터링·대시보드·알림 제공.

```
[App] --READ/WRITE--> [Neo4j Primary] --replicate--> [Neo4j Secondary (GDS/analytics, READ)]
```

### Initial data load

- 수백만~1억 레코드 수준은 transactional 방식(LOAD CSV) — index 갱신·tx log 오버헤드 있으나 유연하고 cluster 복제 보장.
- 프로덕션은 LOAD CSV 대신 Neo4j 프로토콜 client로 ingestion(DB heap 사용 방지). 대용량은 neo4j-admin import(오프라인 CSV) 사용.

### Incremental data load

- 스트리밍 데이터는 Apache Kafka 등 프레임워크 활용. Java·JavaScript·.NET·Python client로 ingestion.
- managed transactional function을 활용해 cluster topology 변경 시 driver가 재시도하도록 함.

### Graph augmentation

- article·customer augmentation(Spring Boot) 후 KNN similarity·community detection을 자동화 필요.
- 새 데이터 ingestion 시 augmentation으로 embedding 생성 → GDS 알고리즘 트리거. ML 파이프라인은 필요에 따라 더 복잡해질 수 있다.
- 추천을 Cypher query로 검증 후 on-demand 추천 제공 앱 구축. Spring Boot의 packaging·actuator로 프로덕션 배포·모니터링.
- augmentation 앱은 실패 지점에서 재시작 가능하도록 설계됨.

---

## Summary (핵심 정리)

- Haystack GenAI 챗봇을 로컬에서 Google Cloud Run에 serverless로 배포(컨테이너화·서비스 설정·배포·검증)하는 전 과정을 익혔다.
- knowledge graph·vector search·GenAI workflow(Haystack·Neo4j)·클라우드 배포까지 end-to-end blueprint를 완성했다.
- 프로덕션을 위한 Primary/Secondary 아키텍처와 데이터 ingestion·augmentation 자동화 고려사항을 제시하며 책의 여정을 마무리한다.
