# 01. Introduction to Model Serving and Optimization

## 챕터 개요 (3줄 요약)
- model serving은 학습된 모델을 production에서 API/service로 노출해 새 입력에 대한 inference를 생성하는 과정으로, 모델의 비즈니스 가치는 적절한 latency·reliability·cost로 "배달"될 때 실현된다.
- 모델은 정적 데이터가 아니라 model data(weights/bias/config) + model architecture + execution code로 구성된 실행 가능한 프로그램으로 다룬다.
- 특히 LLM은 naive 배포 시 latency 폭증·throughput 정체·cost 선형 증가가 발생하므로 serving optimization이 선택이 아닌 필수다.

---

## 1. Anatomy of a Model
> 모델을 black-box 실행 파일로 보고 세 요소 — model data(weights·bias·embeddings·max_batch_size 등 metadata), architecture(layer 구조·연결·연산), execution code(architecture 초기화·weight 로드·prediction 실행) — 로 구성한다고 정의한다.
- architecture와 weight를 별도 파일로 분리 저장하면 partial loading(nonmatching keys skip), fine-tuning, layer 추가, 점진적 architecture update에 유리하다.
- TensorFlow/PyTorch는 API가 달라도 packaging·structure 원칙은 거의 동일하다.

---

## 2. Model Lifecycle: From Training to Serving
> ML lifecycle은 data collection → training/fine-tuning → evaluation → deployment → serving → optimization/iteration으로 이어지며, serving이 이 책의 초점이다.
- training과 production 운영은 제약·목표·엔지니어링 과제가 완전히 다르다 — 모델 가치는 배포·serving 단계에서 실현된다.

---

## 3. What Is Model Serving?
> model serving은 ML 모델을 production에 배포해 input 수신 → inference 실행 → 결과 반환을 efficient·scalable·reliable하게 수행하도록 SW/HW 인프라를 구성하는 것이다.
- 배포 위치: on-device(local), on-premises(in-cluster), on-cloud(remote) — 예: 창고 robot(YOLO), 사내 semantic search(Sentence-BERT), 챗봇(SageMaker/OpenAI).
- 핵심 관심사: deployment, scalability/availability, latency, monitoring(drift·health), versioning, security, cost-to-serve(가장 결정적 지표).
- LLM serving은 Transformer 등 모델 내부 이해가 필수 — 모든 최적화는 LLM 실행 bottleneck 해소를 목표로 하기 때문이다.

---

## 4. Why Study Model Serving? (자체 serving stack의 가치)
> cloud vendor(SageMaker/Bedrock)나 foundation LLM(OpenAI 등)을 쓰더라도, 통합 노력·cost trade-off·data privacy 때문에 serving 원리 이해가 필요하다.
- cost: open source LLM(DeepSeek)을 EC2에서 직접 운영하면 managed 대비 30~60% 절감, reserved instance로 추가 할인 가능.
- 실무 챗봇은 보통 작은 intent-classification 모델 + embedding 모델 + LLM을 조합해 비용을 최적화한다.
- privacy/security(GDPR·HIPAA)와 fine-tuned 모델 serving 비용(OpenAI는 50% 추가 과금) 때문에 in-house가 유리할 수 있다.
- 단, in-house는 HW capital·전문 인력 등 선투자가 필요 — build vs outsource는 프로젝트 성숙도·규모·운영 역량에 따라 결정. one-size-fits-all 아키텍처는 없다.

---

## 5. Why Optimize Model Serving (Especially for LLMs)?
> serving optimization은 latency 감소·throughput 증가·resource 활용 최적화로 cost를 통제하면서 효율을 극대화하는 과정이다.
- LLM 요청은 traditional keyword search보다 ~10배 비쌀 수 있어(Hennessy, 2023) 최적화가 거의 필수다.
- 목표는 단순 성능 향상이 아니라 sustainable cost로 latency·throughput SLA를 충족하는 것.

---

## 6. Example: vLLM으로 LLM Throughput 개선
> training framework와 serving framework는 목표가 다르다 — training은 backprop·weight update 중심 고throughput, serving은 forward-only 저latency 중심.
- vLLM(UC Berkeley Sky Lab)은 PagedAttention·KV Cache·bf16·tensor-parallel 등 knob 제공.
- 2023 실험: PagedAttention으로 HF Transformers 대비 최대 24배, TGI 대비 최대 3.5배 throughput 달성.
- 저자 경험: DeepSeek R1을 FP8 MLA kernel + batch size 증가만으로 38→600 TPS(15배) 향상.

```
vLLM serving config (H100):
--model openai/gpt-oss-20b
--dtype bf16
--gpu-memory-utilization 0.9
--max-num-seqs 16
--max-num-batched-tokens 16384
--tensor-parallel-size 2
```

---

## 7. Model Serving Paradigms — On-Device (Edge) Serving
> 모델을 user-side device에서 직접 실행해 network 없이 real-time 처리하는 방식으로, cost-effective·efficient·private·personalized 이점을 가진다.
- 핵심 컴포넌트: model runtime(LiteRT·ONNX Runtime·Core ML; HW 추상화 + delegate로 GPU offload) + model wrapper(전처리·로드·실행·후처리 캡슐화).
- 배포 흐름: 학습 포맷 → runtime 포맷 변환(.tflite 등) → numerical accuracy 검증 → 성능 측정 → app에 packaging.
- 적합 케이스: privacy-first, ultra-low-latency(AR/VR·robotics), 연결 불안정 환경, IoT/smart city.
- 제약: compute/storage 한계, power 소비, 빈번한 update 어려움, HW(NPU/CPU) 비일관성.

---

## 8. Single-Model Service
> 각 모델·버전을 dedicated web service로 배포해 HTTP/gRPC prediction API를 노출하는, 가장 널리 쓰이는 cloud serving 패턴.
- 컨테이너 3요소: API-Server, Model Management(다운로드·로드), Inference Backend(TF Serving·TorchServe·vLLM 등). containerization(Docker/K8s)이 기반.
- routing: round-robin은 요청별 처리 비용 차이를 무시 → weighted round robin, least connections, least response time, dynamic load balancing 사용.
- scaling: horizontal(scale-out, K8s HPA autoscaling) + vertical(scale-up, 대형 모델용 다중 GPU). intra-node(단일 머신 다중 GPU)를 inter-node보다 우선(network overhead 회피).
- 장점: 최고 성능·격리·독립 scaling·디버깅 용이·장애 격리. 단점: 모델 수 폭증 시 resource 비효율·cost(1,000개 서비스 운영 부담).

---

## 9. Multi-Model Service
> 다수 모델을 한 컨테이너에 co-host해 GPU/CPU·memory를 공유하고 traffic 기반으로 동적 load/unload하여 cost를 최적화하는 패턴.
- on-demand 로딩: 요청 시 로드, 비활성 시 unload. LRU cache로 메모리 임계치(예: 80%) 초과 시 least-used 모델 제거.
- model server inference backend는 여러 backend(TF·ONNX·PyTorch)를 내부 보유 — NVIDIA Triton이 대표 솔루션(다양한 포맷 통합 API).
- routing/scaling: 모델 metadata에 replica 속성 추가 + route map으로 "이미 로드된 컨테이너"로 라우팅(cold start·model swapping 회피), hot 모델은 replica 증설(bin packing).
- 부적합: 모델이 GPU에 안 들어갈 때, 항상 로드된 저latency 고traffic 모델, 보안 정책 상이, 운영 복잡도 높을 때.

---

## 10. Model Serving Platforms
> 비즈니스 성장으로 multi-model 협업과 compute 최적화가 필요해지면, single/multi-model을 단순 결합한 수준을 넘어선 platform이 필요하다.
- resource group으로 app별 workload를 격리하고 CPU/GPU/memory quota를 개별 할당.
- graph execution(Airflow·Ray)으로 multistep inference workflow 지원 — 각 step에서 routing 컴포넌트가 적절한 serving group·service 호출.
- 실무 platform은 security·access control·monitoring·DevOps 통합 등이 추가되나 핵심 serving 개념에 집중하기 위해 도식에서는 생략.

---

## Summary (핵심 정리)
- 모델은 model data + architecture + execution code로 구성된 실행 가능한 프로그램이며, serving은 이를 production에 배포해 efficient·scalable·reliable하게 inference를 제공하는 엔지니어링 중심 분야다.
- 배포 위치(on-device/on-premises/cloud)와 패턴(single-model → multi-model → platform)은 latency·cost·scalability·복잡도 trade-off에 따라 선택하며, one-size-fits-all 아키텍처는 없다.
- single-model은 격리·성능·독립 scaling이 강점, multi-model은 자원 공유로 cost 효율이 강점, platform은 multi-model 협업과 quota 기반 resource 최적화를 제공한다.
- LLM serving은 cost가 핵심 동인 — vLLM의 PagedAttention/KV Cache, SGLang의 Radix Attention 등으로 추가 인프라 비용 없이 6배 throughput·1/3 latency 달성 가능하므로 optimization은 필수다.
