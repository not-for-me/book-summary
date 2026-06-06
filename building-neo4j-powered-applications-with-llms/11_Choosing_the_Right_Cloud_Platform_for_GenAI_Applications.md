# 11. Choosing the Right Cloud Platform for GenAI Applications

## 챕터 개요 (3줄 요약)

- GenAI 애플리케이션 배포 시 클라우드 플랫폼 선택을 위한 핵심 고려사항을 종합적으로 다룬다.
- scalability·성능, 비용·가격 모델, 보안·컴플라이언스 측면에서 Google Cloud·AWS·Azure를 비교한다.
- 요구사항 정의부터 trade-off 평가까지 단계별 의사결정 프레임워크를 제시한다.

---

## 1. Understanding cloud computing options for GenAI applications

> 클라우드는 GenAI의 막대한 연산 요구를 충족하는 필수 기반이며, 특화된 AI 서비스를 제공한다.

- 클라우드 장점: 동적 scalability(수천 개 GPU/TPU), pay-as-you-go 비용 효율, 글로벌 접근성(저지연), 엔터프라이즈급 보안·가용성, 협업·통합.
- 특화 AI 서비스: pre-trained 모델·API(텍스트 생성·음성·이미지), custom AI 학습·fine-tuning, managed AI workflow(전처리~배포 자동화), 데이터 서비스 통합.

### 주요 제공자 AI 역량 비교 (Figure 11.1)

- Google Cloud(Vertex AI): Gemini·Model Garden, AutoML·fine-tuning(PyTorch/TensorFlow/JAX), Vertex AI pipelines, BigQuery ML.
- AWS(SageMaker): Bedrock(Anthropic Claude·Meta Llama·AI21), custom training, SageMaker pipelines·Step Functions, S3·Redshift ML·Glue.
- Azure(Azure AI/ML): Model Catalog(OpenAI GPT-4·Meta Llama), AutoML·ML pipelines, Cognitive Services, Synapse·Data Lake·Databricks.

---

## 2. Picking a cloud platform: key considerations

### Scalability and performance

> GenAI는 자원 집약적이라 동적 확장과 GPU/TPU 같은 고성능 하드웨어가 성공에 필수다.

- Elastic scaling: autoscaling으로 워크로드에 따라 자원 자동 조정, 미사용 자원 비용 방지.
- High-performance hardware: GPU·TPU·FPGA로 학습·추론 가속.
- Regional availability·latency: 글로벌 사용자에 저지연 제공.
- Batch·real-time processing: 대량 학습과 실시간 챗봇 응답 모두 효율적 처리.
- 비교(Figure 11.2): Google(TPU·NVIDIA A100/H100·Axion, 35+ region), AWS(Inferentia·Trainium, 32+ region 최대 footprint), Azure(GPU·FPGA, 60+ region·강력한 hybrid).

### Cost and pricing models

> AI 워크로드는 비용이 빠르게 증가할 수 있어 적절한 가격 모델·절감 전략 선택이 중요하다.

- Pay-as-you-go: 사용한 만큼 과금(e-commerce 성수기 확장 예시).
- Reserved instances: 1~3년 약정으로 최대 75% 할인(예측 가능한 워크로드, 의료 영상 진단 예시).
- Spot instances: 미사용 용량을 최대 90% 저렴하게(중단 허용 batch 학습).
- Data storage costs: hot/warm/cold 티어로 접근 빈도별 비용 최적화.
- Data transfer costs: region 간·egress 비용 주의(금융 사기 탐지 cross-region 예시).
- 비용 관리 도구: AWS Cost Explorer, Google Billing Reports, Azure Cost Management, free tier·trial credit.
- 최적화: right-sizing(Compute Optimizer·Advisor·autoscaling·serverless·컨테이너), data lifecycle management, centralized billing.

### Security and compliance

> GenAI는 민감 데이터를 다루므로 보안과 규제 준수가 필수다. 일부는 기본 활성화, 일부는 수동 설정 필요.

- Data encryption: in transit(TLS/SSL), at rest(AES-256 기본, CMEK/CMK 등 customer-managed key).
- IAM: RBAC(Admin/Developer/Viewer), MFA, SSO·federated auth(AAD·AWS IAM Identity Center·Google IAM), least privilege.
- Network security: VPC, firewall·private endpoint(PrivateLink·Private Link·Private Service Connect), logging·monitoring(VPC Flow Logs·Network Watcher).
- Threat detection·incident response: ML 기반 anomaly detection, 자동 remediation, SIEM 연동(AWS GuardDuty·Azure Security Center).
- 규제 표준: HIPAA(의료 PHI), GDPR(EU 개인정보), SOC 2(데이터 보안). data residency·sovereignty(지역 저장), auditability(CloudTrail·Azure Monitor Logs).
- 인증(Figure 11.6): ISO 27001, PCI DSS, FedRAMP, SOC 2, HIPAA, GDPR — 3사 모두 준수(HIPAA는 수동 설정 필요).

### Best practices

- 정기 보안 감사, 자동 compliance 체크, shared responsibility model 이해, data minimization.
- Cloud Security Alliance(CSA) STAR로 제공자 보안 통제·인증 비교 가능.

### Key takeaways

- Google Cloud: AI 분석·end-to-end ML workflow·데이터 통합 강점.
- AWS: 탁월한 scalability·비용 절감 옵션, 고볼륨·유연 워크로드에 적합.
- Microsoft Azure: hybrid cloud 지원·엔터프라이즈급 AI 서비스 강점.

---

## 3. Decision-making framework for selecting your cloud platform

> 프로젝트 요구·우선순위·제약에 따라 단계별로 플랫폼을 평가하는 구조화된 방법론.

- 요구사항·우선순위 정의: 성능(GPU/TPU), scalability(steady vs fluctuating), AI 특화 기능, 예산, compliance·보안.
- 후보 제공자 shortlist: AI 서비스(Vertex AI·SageMaker·Cognitive Services), 비용 모델, 통합·생태계(TensorFlow·PyTorch), regional availability.
- Trade-off 평가: 성능 vs 비용, 유연성 vs 특화, 지원·신뢰성.
- 의사결정 도구 활용: cost calculator, trial·free tier, 성능 benchmark.
- Future-proof: 장기 비용 효율(reserved·hybrid), 진화하는 AI 요구, vendor lock-in 위험(portability).
- 데이터 기반 결정: 비용·성능·scalability·compliance·기능을 가중치 매긴 comparison matrix로 최적 플랫폼 선정.

```
Define requirements -> Shortlist providers -> Evaluate trade-offs
   -> Leverage tools -> Future-proof -> Data-driven decision (weighted matrix)
```

---

## Summary (핵심 정리)

- 클라우드의 scalability·성능 역할과 특화 AI 서비스의 중요성을 강조했다.
- 비용·보안·compliance·기능 측면에서 Google Cloud·AWS·Azure의 강점·가격 모델을 분석했다.
- 다음 Chapter 12에서는 Google Cloud에 GenAI 애플리케이션을 실제 배포하는 실습 워크플로를 다룬다.
