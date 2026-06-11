# 07. Model Deployment and Prediction Service

## 챕터 개요 (3줄 요약)
- 배포의 어려움은 ML이 아니라 engineering 문제이며, deployment에 대한 흔한 myth들을 먼저 깨뜨린다.
- prediction 제공은 batch prediction(asynchronous, high throughput)과 online prediction(synchronous, low latency)으로 나뉘고 streaming feature 사용 여부가 핵심이다.
- inference latency를 줄이는 model compression(quantization 등)과 cloud vs edge 선택, hardware별 compiler·IR·model optimization을 다룬다.

---

## 1. ML Deployment Myths
> **Myth 1**: 한 번에 1~2개 model만 배포 → 실제론 기능·국가별 수백~수천 model(Uber 수천, Booking 150+).
> **Myth 2**: 가만두면 성능 유지 → software rot + data distribution shift로 학습 직후 최고, 시간이 지날수록 저하.
> **Myth 3**: model을 자주 update 안 해도 됨 → "얼마나 자주 update 할 수 있나"가 옳은 질문(Weibo 10분 cycle).
> **Myth 4**: 대부분 scale 걱정 불필요 → 통계적으로 ML engineer는 100명+ 기업에서 일할 가능성 높아 scalability 필요.

## 2. Batch vs Online Prediction
> 3가지 mode: ① batch prediction(batch feature만), ② online prediction(batch feature만, 예: precomputed embedding), ③ online prediction(batch+streaming feature = streaming prediction).
> **online**(on-demand, synchronous): 요청 도착 즉시 생성(Google Translate). REST API/HTTP. low latency 최적화.
> **batch**(asynchronous): 주기적/trigger 시 생성·저장 후 fetch(Netflix 추천 4시간마다). high throughput 최적화.
> streaming feature(streaming data 전용) vs online feature(메모리 batch feature 포함, 더 일반적). 둘은 배타적이지 않고 hybrid(인기 query 사전계산 + 비인기 online) 가능.

```
batch (async):  periodic, high throughput, recommender systems
online (sync):  on request, low latency, fraud detection
```

## 3. From Batch to Online Prediction
> online의 문제는 생성 시간이 긴 것 → batch는 사전계산으로 latency 회피(복잡 model의 트릭). 단 batch는 사용자 선호 변화에 둔감(Netflix가 comedy 검색 즉시 반영 못함), 요청을 사전에 알아야 함(번역처럼 예측 불가 query엔 부적합).
> online 필수: high-frequency trading, autonomous vehicle, voice assistant, face/fingerprint 잠금해제, fall detection, fraud detection.
> online 전환 위해 ① (near) real-time pipeline(real-time transport + stream computation engine), ② 충분히 빠른 model(consumer 앱은 ms 수준) 필요.

## 4. Unifying Batch & Streaming Pipeline
> arrival time 예측 예: 동일 feature(지난 5분 평균 속도)가 training 시엔 batch, inference 시엔 streaming(sliding window)으로 계산 → 두 pipeline 불일치가 production bug의 흔한 원인(특히 다른 팀 유지 시).
> Uber·Weibo는 Apache Flink로 batch/stream pipeline 통합, feature store로 일관성 보장(Ch10).

## 5. Model Compression
> latency 감소 3접근: 더 빠른 inference, 더 작은 model(compression), 더 빠른 hardware. 4대 기법:
> **low-rank factorization**: 고차원 tensor를 저차원으로(compact convolutional filter). SqueezeNet(AlexNet 정확도, 50x 적은 param), MobileNet(K×K×C를 depthwise+pointwise로 분해). model 특화·아키텍처 지식 필요.
> **knowledge distillation**: 작은 student가 큰 teacher 모방(DistilBERT: 40% 작고 97% 성능 유지, 60% 빠름). 아키텍처 무관하나 teacher 의존.
> **pruning**: 중요도 낮은 node 제거 또는 parameter를 0으로(sparse화). nonzero param 90%+ 감소 가능.
> **quantization**: 가장 일반적·범용. 적은 bit로 parameter 표현(32→16 half precision, 8-bit fixed point, 1-bit binary). memory↓·연산 속도↑·batch size↑. rounding error 위험. training 중(quantization aware) 또는 post-training. Roblox 사례: quantization으로 latency 7x↓, throughput 8x↑.

## 6. ML on Cloud vs Edge
> **cloud**: 설정 쉬우나 비용 큼(기업 연 수억 달러), 인터넷 의존, network latency, 데이터 유출 위험.
> **edge**(consumer device): 비용↓, 인터넷 없이 작동, network latency 회피, 민감 데이터 로컬 처리(GDPR 준수 용이). 단 device가 충분한 compute·memory·battery 필요(폰에서 full BERT는 배터리 소모).
> 2025년 edge device 300억+ 전망, 기업들이 AI chip 경쟁.

## 7. Compiling & Optimizing for Edge
> framework(TF/PyTorch)가 hardware backend에서 돌려면 vendor 지원 필요(CPU=scalar, GPU=1D vector, TPU=2D tensor 등 memory layout·compute primitive 상이).
> **IR(intermediate representation)**: framework와 hardware 사이 중간자. compiler가 high→low level IR 생성 후 native code로 "lowering"(high-level IR=computation graph).

```
framework code -> high-level IR -> low-level IR -> machine code
```

## 8. Model Optimization
> lowered code가 비효율적일 수 있음(data locality·cache·vector/parallel 미활용). 최적화는 local(operator)과 global(전체 graph).
> local 기법: **vectorization**(연속 메모리 동시 처리), **parallelization**(독립 chunk 분할), **loop tiling**(memory layout 활용, hardware 의존), **operator fusion**(여러 operator 결합으로 중복 memory access 제거). graph의 vertical/horizontal fusion으로 더 큰 가속.
> **ML로 ML 최적화**: 수작업 heuristic은 비최적·비적응적. autoTVM(TVM)은 computation graph를 subgraph로 나눠 cost model로 최적 path 예측, runtime 데이터로 학습해 hardware 적응. 느리지만 1회성·cache 가능(cuDNN 대비 ResNet-50 가속).

## 9. ML in Browsers
> browser에서 실행하면 어떤 device든(MacBook/iPhone/Android) chip 무관하게 동작. JavaScript(TensorFlow.js)는 느림.
> **WebAssembly(WASM)**: browser에서 executable 실행 표준. 모델을 WASM으로 compile → JS에서 사용. 93% device 지원. 단 native 대비 평균 45~55% 느림.

---

## Summary (핵심 정리)
- model 배포는 ML이 아닌 engineering 과제이며, online prediction(사용자 변화에 반응, latency 우려)과 batch prediction(빠른 응답, 유연성↓)을 use case에 맞게 선택한다.
- inference latency는 model compression(quantization이 가장 범용·효과적, 그 외 low-rank/distillation/pruning)으로 줄이고, batch/streaming pipeline 통합으로 train-inference 불일치 bug를 막는다.
- cloud는 설정 쉽지만 비용·latency·privacy 문제가 있고 edge는 device 성능이 필요하며, hardware별 실행은 IR·compiler·model optimization(operator fusion, autoTVM)·WASM으로 해결한다.
