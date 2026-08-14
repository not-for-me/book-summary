# 07. Data Analysis and Preparation

## 챕터 개요 (3줄 요약)

- Kubeflow notebook으로 대화형 data 분석을 하고, KFP로 재현 가능한 data preparation pipeline을 구축한다("garbage in, garbage out").
- 두 capstone project(ID card detector = object detection, movie recommender = tabular)의 data prep 단계를 각각 component로 나눠 만든다.
- KFP component 간 data passing 규칙(simple value vs 대용량 path/Artifact)과 KFP v2 artifact type을 상세히 다룬다.

---

## 7.1 Data analysis (Kubeflow notebooks)

> 대규모 dataset은 data 가까이서 처리하되 GPU 등 고비용 resource를 ad hoc으로 spin up/down해야 한다.

- Kubeflow notebook은 Jupyter/VS Code/RStudio를 pipeline과 같은 cluster에서 제공 → deployment 환경과 근접, 동일 base image·CI/CD로 version 관리.
- **7.1.1 launch**: Notebooks > NEW NOTEBOOK. K8s Namespace(resource quota·volume·secret), image 선택, resource limit(너무 낮게 X), GPU/TPU 선택(비싸므로 미사용 시 stop life cycle rule).
- **7.1.2 volumes**: workspace volume 자동 mount(`/home/jovyan`, 영구 저장), data volume은 기존 PVC를 runtime mount.
- **7.1.3 config/affinity**: **PodDefault**로 env var 주입(예: MLFLOW_TRACKING_URI). **affinity**(특정 node 유인)/**toleration**(taint node 허용) — GPU node 접근 제어. notebook은 **CRD**(notebooks.kubeflow.org), 수정 시 redeploy(비영구 data 삭제).
- **7.1.4 menu 커스터마이즈**: `jupyter-web-app-config` ConfigMap 편집 — 사용자에게 보일 image 목록·기본값·readOnly, affinity option(small-cpu/xl-cpu 등).
- **7.1.5 custom image**: dependency + runtime + config layer. multistage Dockerfile로 code-server/Jupyter image 빌드.

---

## 7.2 Data passing (KFP)

> 각 component는 별도 K8s pod에서 격리 실행 → data 공유 mechanism 필요.

### 7.2.1 Scenario 1: 단순 값 전달

- `@dsl.component`로 함수를 component화. import는 **함수 내부**에 위치(dependency 캡슐화). `packages_to_install`(custom package), `base_image` 지정.
- 반환값은 `.output` 속성으로 downstream에 전달 → KFP가 dependency·serialization 자동 처리.
- 컴파일 시 생성 YAML은 code injection으로 pip 설치 + ephemeral component 실행 로직을 삽입.

```python
@dsl.component(packages_to_install=["pyjokes"], base_image="python:3.11")
def generate_joke() -> str:
    import pyjokes
    return pyjokes.get_joke()

@dsl.pipeline(name="joke_pipeline")
def pipeline():
    j = generate_joke()
    c = count_words(input=j.output)
    output_result(count=c.output)
```

### 7.2.2 Scenario 2: 대용량 data는 path 전달

- component는 pod의 memory limit 초과 시 **OOMKilled**. 큰 data는 raw value 대신 file/directory **path**를 전달.
- KFP v2는 `Output[Artifact]`/`Input[Artifact]`로 directory 제공, `.path` 속성으로 접근. (v1의 InputPath/OutputPath 네이밍 규칙 대체 — 더 type-safe).

```python
@dsl.component(...)
def generate_joke(num_of_jokes: int, output_jokes: Output[Artifact]):
    os.makedirs(output_jokes.path, exist_ok=True)
    # write to os.path.join(output_jokes.path, "jokes.txt")

@dsl.component
def count_words(jokes_file: Input[Artifact]) -> int:
    # read from os.path.join(jokes_file.path, "jokes.txt")
```

### 7.2.3 KFP v2 artifact types (8종)

- **Dataset**: 모든 ML data(train/val/test, feature set, preprocessed).
- **Model**: trained model·checkpoint·weight, framework metadata·lineage.
- **Metrics**: key-value 수치 metric. **ClassificationMetrics**(ROC, confusion matrix, UI 렌더링) + **SlicedClassificationMetrics**.
- **HTML**: 인터랙티브 시각화·리포트. **Markdown**: 문서·model card·README.
- **Artifact**(base): generic file·config·중간 결과. metadata tracking + 표준 저장·UI 렌더링.

---

## 7.3 Data preparation in action

> 두 project의 training pipeline 기초 = data prep 단계(download + split). 각 단계 = 1 component.

### 7.3.1 Object detection (ID card)

- **YOLO 형식**: config에 root path, train/val/test image 경로, class names(0: id_card). label은 대응 `labels/` 폴더(images/id0042.png → labels/id0042.txt).
- dataset: **MIDV-500**(ID card만 필터). Ultralytics YOLO 형식으로 변환.
- **Download component**(4 step): (1) remote(Box URL, ~10GB)에서 tqdm progress bar로 chunk download, (2) tarfile로 uncompress, (3) boto3로 **MinIO**(S3 호환, Kubeflow 내장) bucket 생성, (4) images/labels upload. → `@dsl.component`로 wrap, `Output[Dataset]`로 아티팩트 관리.
- 주의: production에서 default password(minio/minio123) 금지 → Vault + K8s sidecar 권장.
- **Split component**: scikit-learn `train_test_split`로 75/15/10(train/val/test) 2단계 분할. `random_state`로 재현성. 각 split을 images/labels 폴더로 정리(shutil.copy2).
- **output_file_contents**: sanity check용 — directory tree 출력.
- **Full pipeline**: download → split → verify. KFP가 output↔input으로 dependency 자동 관리. UI 또는 SDK로 upload·run(experiment로 그룹화). production: log rotation·diskPressure 주의.

```text
[download_dataset] --Output[Dataset]--> [split_dataset] --> [output_file_contents x2]
       (MinIO)              (75/15/10 train/val/test)         (sanity check)
```

### 7.3.2 Movie recommender (tabular)

- dataset: **MovieLens 25M**(GroupLens, 25M ratings, 62K movies, 162K users — license 준수).
- notebook에서 6개 component 구축: download → unzip → CSV→Parquet → split → MinIO upload → QA.
- **containerized Python component**: `target_image` 지정 시 build time에 image 빌드·push(runtime pip 설치 X, 빠른 startup, 외부 module import 가능). Docker daemon 필요. lightweight와 full container 중간.
- **Parquet 변환**: tabular은 columnar 포맷이 유리(sharding, 작은 size, 성능). pandas + fastparquet. `OutputPath`는 Kubeflow가 이름 지정(/tmp) → 필요 시 upload_file_name으로 rename. Argo가 /tmp를 zip해 MinIO에 업로드(component 간 persistence).
- **QA component**: 파일 4개 존재 + train ~75% 검증. **PyArrow**로 MinIO에서 직접 read. size mismatch·missing file 조기 감지. production에선 size·schema·content 모든 가정 검증.
- **Full pipeline**: SDK(`kfp.Client()`)로 upload·run. cluster 내 notebook은 env var로 credential 자동 처리(외부는 endpoint 지정). `.after()`로 수동 순서 강제, caching option 제어.

---

## Summary (핵심 정리)

- ML workflow는 data 근접 처리 + GPU 등 ad hoc 특수 resource 접근이 필요하며, Kubeflow notebook이 deployment와 일관된 Jupyter 환경을 같은 cluster에 제공한다.
- notebook은 base image 공유·CI 통합으로 versioning·CI/CD를 streamline한다.
- component는 함수로 만들며 import는 함수 내부, dependency는 `packages_to_install`로 지정하고 `@dsl.pipeline`으로 조합한다.
- 대용량 data는 raw 대신 path/Artifact로 전달해 OOMKilled를 피하며, KFP v2의 8개 artifact type이 type-safe한 data lineage를 제공한다.
- Parquet 변환·dataset split 등 효율적 data handling을 익혔다. 다음 챕터(8장)에서는 이 data로 object detector·movie recommender model training을 다룬다.