# 05. Extracting Domain-Specific Knowledge from Unstructured Data

## 챕터 개요 (3줄 요약)

- 이메일·논문·뉴스 등 비정형 데이터에서 지식을 추출하려면 지식 표현(knowledge representation)과 지식 학습(knowledge learning, NLP/LLM)이라는 두 과제를 풀어야 한다.
- Rockefeller Archive Center(RAC)의 역사적 일기 문서를 사례로, 명명 엔티티 인식(NER)과 관계 추출(RE)을 LLM으로 수행한다.
- 전통 NLP 대비 LLM은 전이학습·프롬프트 엔지니어링으로 낮은 초기 비용에 더 깨끗한 KG를 만들지만, 예측 속도·인프라·보안 측면 트레이드오프가 있다.

---

## 1. The Archives Challenge

> 아카이브는 책·보고서·회의록 등 다양한 복잡도와 문체의 비정형 역사·현재 데이터를 포괄한다.

- RAC는 40개 이상 자선재단(록펠러·포드 등)의 기록을 보유한 자선·연구 부문 연구센터다.
- 록펠러 재단의 프로그램 담당자(program officer)는 연구자 네트워크를 구축하며 회의·전화·만찬을 약어·전문 용어로 타자기에 급히 기록한 일기를 남겼다.
- 이 일기를 적절히 마이닝·모델링하면 자금 지원 패턴, 영향력 있는 과학자의 추천 영향, 내부 논의 횟수, 시간에 따른 하위 분야 자금 추세 같은 질문에 답할 수 있다.
- OCR(광학 문자 인식), NER(Named Entity Recognition), RE(Relation Extraction) 같은 최신 ML 모델에 의존하는 단계로 비정형 텍스트를 구조화 지식으로 변환한다.

---

## 2. Key Concepts of Knowledge Extraction

> 텍스트를 KG로 변환하는 핵심은 텍스트 내 엔티티를 식별하고 이를 연결하는 관계를 추출하는 것이다.

### 5.2.1 Recognizing Named Entities (NER)

- NER은 원시 텍스트에서 명명 엔티티 언급을 식별해 사전 정의 범주로 분류하는 ML 분류 시스템이다.
- 비즈니스 활용: 다양한 소스 문서 연결, 정보 관리·데이터 거버넌스 개선, 데이터 컴플라이언스 기반 마련, 검색 향상, 원인-결과 연관.
- 일반 범주(Person, Location, Organization)만으로는 부족하며, Warren Weaver 일기에서는 대화 주제(Topic: 연구 분야·기술·질병 하위 타입)를 인식해야 한다.
- 사전 기반 NER로는 'aerological research' 같은 도메인 주제를 식별 못 하므로, 맞춤 NER 모델 훈련이나 LLM 활용이 필요하다.

### 5.2.2 Extracting Relations (RE)

- RE는 텍스트 내 엔티티 쌍 사이의 의미적 관계를 식별하는 것으로(예: PERSON이 ORGANIZATION에 고용됨), 이 연결성 포착이 진정한 KG를 만든다.
- 관계 모델링 방식은 다양하나(WORKS_FOR vs EMPLOYS) 문서 전체에서 일관성 유지가 중요하다.

---

## 3. Building KGs with Large Language Models

> LLM은 전이학습으로 일반 작업(마스킹 토큰 예측)에서 배운 패턴을 RE 같은 작업에 재사용해, 적은 라벨 데이터로 KG를 구축한다.

- LLM의 핵심은 모델 복잡도(파라미터 수)와 학습 코퍼스의 규모·품질이며, 데이터 중심(data-centric) 패러다임이 부상했다.
- 가장 중요한 한계는 환각(hallucination): 학습 데이터에 근거가 없을 때 사실을 지어내는 경향(예: NASA SLS 비용을 무작위로 답함).

### 5.3.1 Using LLMs

- LLM은 프롬프트 기반 추론(out of the box)으로 쓰거나, 도메인 특화 정확·안정 출력을 위해 파인튜닝(fine-tuning)할 수 있다.
- 본 장은 텍스트 완성(text completion)에 관심을 두며, 데이터 탐색→프롬프트 엔지니어링→(만족 시) KG 생성 또는 (복잡 시) 파인튜닝 경로를 따른다.
- 사람이 이름·초기자(예: "R.")로 다르게 참조되는 문제(coreference)를 해소하도록 시스템을 설계해야 한다.

### 5.3.2 Prompt Engineering Examples

- 프롬프트 v1(엔티티·관계 모두 추출)은 인상적이나 동일 개념에 과도하게 세분화된 관계 타입(specializes in, is measuring, is interested in 등)과 불안정한 예측을 낳았다.
- LLM의 생성적 특성 덕에 coreference 해소가 암묵적으로 처리되고, 약어 제목("Prof. Chem.")이 전체형("Professor of Chemistry")으로 확장된다.
- 프롬프트 v2(관심 엔티티 클래스·관계 타입 목록과 약어 규칙 명시)는 더 정규화된 출력(occupation 단일 라벨, works on 관계)을 유도했다.
- 프롬프트 v3(더 복잡하고 대표성 있는 예시 추가, student of·occupation 강조)에서 엔티티·관계 클래스 할당의 높은 안정성과 모든 관심 관계의 정확한 식별을 달성했다.
- 실제 프로젝트에서는 출력을 JSON으로 바꿔 각 엔티티·관계에 속성(예: TALKED_ABOUT의 감정, WORK_FOR의 직함)을 부여했다.

```
  Text (diary) --[Prompt-based LLM (NER+RE)]--> [ENTITY1, TYPE, RELATION, ENTITY2, TYPE]
                              | (if too complex)
                              v
                       Fine-tuning --> domain-specialized model --> KG
```

### 5.3.3 Prompt Engineering Guidelines

- 작업 설명·도메인 특화 안내, 관심 엔티티·관계 클래스 명명(용어 선택이 중요: Topic→Occupation 개명으로 결과 개선), 복잡·대표성 있는 예시 제공.
- LLM 구성: temperature 실험(사실 추출은 0, 창의적 작업은 높게), 예측 안정성 테스트, 프롬프트 단위 테스트, 미니 KG 시각 점검, 정량 평가(precision·recall·F1 또는 LLM-as-judge).
- 야심차게 시도하되(LLM의 깊은 언어 이해 활용) 과도한 엔지니어링은 피하고, 프롬프트가 한계에 이르면 파인튜닝에 시간을 투자한다.

### 5.3.4 Traditional NLP or LLMs?

- 전통 NLP 장점: 빠른 예측 속도, 단순·저렴한 인프라(GPU 불필요), 낮은 예측 비용, 온프레미스 보안 용이.
- 전통 NLP 단점: 높은 사내 전문성 요구, 복잡한 파이프라인(NER·RE·coreference·entity resolution), 데이터 주석 비용·시간이 큼.
- LLM 장점: 빠른 도메인 커스터마이징, 얕은 학습 곡선, 올인원 NLP, 생성적 특성으로 더 깨끗한 KG, 적은 후처리.
- LLM 단점: 느린 예측 속도, GPU 인프라 복잡성, 파인튜닝 비용, 대규모 데이터셋 예측 비용, 온프레미스 보안 배포 비용.
- 결론: 전통 NLP는 보안·대규모 데이터·스트리밍 시나리오에서 여전히 유효하며, LLM은 맞춤 NLP 모델 훈련용 데이터 사전 주석에도 쓸 수 있어 공존 가능하다.

---

## Summary (핵심 정리)

- 비정형 텍스트에서 KG를 만들려면 도메인 특화 명명 엔티티(NER)와 그들 간 관계(RE)를 식별해야 하며, 전통 NLP는 여러 모델을 사슬로 연결해 높은 초기 투자를 요한다.
- LLM(예: GPT)은 반복적 프롬프트 엔지니어링이나 파인튜닝으로 훨씬 낮은 초기 비용에 정확한 도메인 특화 KG를 구축하며, 생성적 특성 덕에 정규화·해소 부담이 적다.
- 전통 NLP와 LLM은 도메인·비즈니스 고려사항에 따라 장단점이 있고 공존하거나 한쪽이 다른 쪽의 훈련 데이터를 준비하는 데 쓰일 수 있다.
