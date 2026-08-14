# 08. Model Training and Validation: Part 1

## 챕터 개요 (3줄 요약)

- YOLO object detection(ID card detector)을 예로 재현 가능·modular한 training·validation component를 만들어 pipeline에 통합한다.
- system-level dependency는 `@dsl.container_component`로, 일반 Python은 `@dsl.component`로 처리하며 base_image를 활용해 dependency hell을 피한다.
- KFP metrics artifact로 mAP 등 metric을 tracking하고, 학습된 model artifact를 다운로드해 "trust but verify"로 수동 검증한다.

---

## 8.1 Training an object detection model

> production training은 재현 가능·modular하며 pipeline infra와 통합되어야 한다(Jupyter는 scale 안 됨).

- training component는 version 관리·환경 일관 실행·data prep pipeline 통합·systematic monitoring 가능해야 함.
- 3가지 책임: (1) train/val/test image·label 다운로드, (2) dataset config(path·label 매칭), (3) 실제 training·evaluation.

### 8.1.1 Custom dataset config

- YOLO는 기본적으로 ID card 미탐지 → YAML config 필요. path, train/val/test image 경로, names(0: id_card). YOLOv8은 image 경로만 지정(label은 인접 `labels/` 폴더).

```python
data = {'path': '/dataset/', 'train': 'train/images', 'val': 'val/images',
        'test': 'test/images', 'names': {0: 'id_card'}}
yaml.dump(data, open('custom_data.yaml', 'w'))
```

### 8.1.2 Training the model

> production은 다양한 시나리오에서 잘 동작하는 robust default config 필요.

- 3 balance 요소: **training efficiency**(batch size·learning rate), **resource utilization**(architecture 선택), **performance requirement**(accuracy vs 시간).
- Ultralytics `YOLO('yolov8n.pt')` → `model.train(data, imgsz=640, epochs, batch, name, project)`. project path에 weights·confusion matrix·PR curve·inference sample 자동 수집.
- **model architecture trade-off**: 작은 model(YOLOv8n)은 edge/real-time(빠름·저메모리), 큰 model은 정확하나 compute 많음. 예: 감시 카메라(속도) vs 자율주행(정확도).
- Nano 선택 이유: compute 절약, 타당성 테스트, iterate·refine.
- 주요 hyperparameter: model, data, epochs(기본 20), batch(기본 4, -1 AutoBatch), imgsz(640), optimizer('auto': >10000 iter는 SGD, 이하 AdamW), pretrained(True).

```python
model = YOLO('yolov8n.pt')
results = model.train(data='custom_data.yaml', imgsz=640,
                      epochs=epochs, batch=batch, name='yolov8n_custom',
                      project=project_path)
```

### 8.1.3 Container components for system dependencies

> Python package로 설치 못 하는 system dependency는 `@dsl.container_component`로 처리.

- Ultralytics image가 `libGL.so.1` 오류 → `apt-get install libgl1-mesa-glx` 필요. `@dsl.component`는 Python만 가능.
- `dsl.ContainerSpec` + `@dsl.container_component`로 임의 Docker command(system 설치) 실행(수동 Dockerfile 불필요, 대신 verbose).
- **4단계**: (1) 독립 실행 가능한 training script(argparse) 작성, (2) `@dsl.container_component`로 image·command·args 지정, (3) heredoc(`cat << 'EOF'`)으로 script 주입·bash positional param($0,$1...)으로 인자 매핑, (4) `@dsl.component`처럼 pipeline에서 사용.

```python
@dsl.container_component
def train_model(epochs: int, batch: int, ...,
                train_dataset: Input[Dataset], model_output: Output[Model], ...):
    return dsl.ContainerSpec(
        image='python:3.11-slim', command=['bash', '-c'],
        args=['apt-get install -y libgl1-mesa-glx && pip install ultralytics ... && '
              'cat << EOF > /train.py ... EOF; python3 /train.py --train-path "$0" ...',
              train_dataset.path, ...])
```

### 8.1.4 Validation component

> validation은 accuracy 측정을 넘어 degradation 조기 감지, segment별 성능, 배포 결정, 이력 유지를 돕는다.

- KFP **metrics output**(Markdown/Plots/raw value)으로 insight 캡처. 여기선 `@dsl.component` 재사용.
- **dependency hell** 회피: requirements.txt로 다 설치 X → prepackaged Docker(Ultralytics base image) + `packages_to_install` 보조.
- `model.val()` 한 줄로 검증. COCO metric(mAP) logging: `metrics.log_metric("map50-95", ...)`, map50, map75.

```python
@dsl.component(base_image="ultralytics/ultralytics:8.0.194-cpu",
               packages_to_install=["minio", "tqdm"])
def validate_model(data_yaml: Input[Artifact], model: Input[Model],
                   validation_dataset: Input[Dataset], metrics: Output[Metrics]):
    m = YOLO(os.path.join(model.path, "best.pt"))
    r = m.val(data=..., imgsz=640, batch=1)
    metrics.log_metric("map50-95", r.box.map)
```

### 8.1.5~8.1.6 Pipeline 구성·실행

- download → split → train → validate. KFP가 output↔input으로 dependency·topology 표현.
- 제공 dataset으로 1 epoch ~1시간(cluster). Ultralytics는 architecture·setup·dataset·progress·speed 정보 출력.
- Run output tab에서 metric 확인, Output artifacts에서 raw metric, Pipelines Overview에서 run별 metric 비교.

### 8.1.7 Validating model artifacts

> "trust, but verify" — model이 실제 동작하는지 수동 검증 필요(넘겨받은 model이 작동 안 한 사례 다수).

- default YOLO(`yolov8n.pt`)로 inference: face를 person, ID card를 book(confidence 0.28)으로 오분류하나 bounding box는 그럭저럭. confidence score(0~1)로 신뢰도 판단·필터.
- KFP 학습 `best.pt`로 교체: 1 epoch에도 ID card 정확 분류(bbox 개선), 5 epoch에 bbox 완전 커버·confidence 향상.
- `best.pt`는 Ultralytics가 validation 최고 성능 weight 저장(100 epoch 중 epoch 4일 수도). 단 너무 적은 epoch(3)면 undertrained.
- 자동화: confidence threshold·bbox 좌표 타당성을 programmatic 검증(tolerance 설정). hyperparameter 튜닝은 iterative — 최소 set부터.

---

## Summary (핵심 정리)

- training data를 model에 전달하는 방법은 version control·lineage를 지원해야 하며, Kubeflow는 direct download 등 여러 방법을 제공한다.
- model architecture는 public metric뿐 아니라 요구사항 대비 trade-off를 이해하고 선택해야 한다.
- training code 작성 시 local access를 가정하지 말고 data access·passing을 고려한다(자동 pipeline 배포 대비).
- metric과 그 가시성(Kubeflow dashboard)은 대규모 team의 협업·feedback loop 단축에 핵심이다.
- model output weight의 빠른 sanity test 능력을 dataflow·저장 전략 설계 시 고려하면 debugging이 쉬워진다.
- 다음 챕터(9장)에서 K8s Persistent Volume, MLflow·TensorBoard 통합, movie recommender training을 다룬다.