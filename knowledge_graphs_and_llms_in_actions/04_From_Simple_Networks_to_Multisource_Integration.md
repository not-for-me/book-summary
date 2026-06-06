# 04. From Simple Networks to Multisource Integration

## 챕터 개요 (3줄 요약)

- 단일 온톨로지를 넘어 여러 구조화 데이터 소스(CSV, RDB, API)를 동질적 그래프로 통합하여 점점 복잡한 KG와 지능형 조언 시스템(IAS)을 구축한다.
- 다중오믹스(multi-omic)·제약(pharmaceutical)·임상(clinical) 세 가지 생의학 KG 응용을 사례로 그래프 모델링·통합·분석 기법을 다룬다.
- WCC·Louvain 같은 일반 클러스터링과 밀도·전도도·DWPC(Degree-Weighted Path Count) 같은 도메인 특화 지표로 KG를 분석하고, LLM이 정량 결과를 해석한다.

---

## 1. Biomedical Knowledge Graphs and Applications

> 기존 생의학 지식을 그래프로 조직하면 신약 재창출, 질병-생체분자 연관 식별, 정밀의료 지원 같은 복잡한 질문에 답할 수 있다.

- 본 장은 구조화 데이터가 주이므로 전통적 데이터 통합 기법이 핵심이고 LLM은 보조 도구 역할에 그친다.
- 응용 유형별로 비즈니스 목표가 다르고 CRISP-DM 모델에 따라 서로 다른 데이터 소스로 구축된다.
- KG는 신약 재창출, 환자 진단, 질병-생체분자 연관, 단백질 기능 식별, 암 유전자 우선순위화, 안전한 약물 추천 등에 활용된다.

---

## 2. Multi-omic Applications of KGs

> 다중오믹스는 게놈(genome, DNA)·전사체(transcriptome, RNA)·단백질체(proteome, protein) 등 여러 "omics" 데이터셋을 함께 활용하는 분석 접근이다.

- 게놈은 생물의 전체 유전 정보, 전사체는 단백질체 합성을 지시하는 RNA 집합(전사 transcription의 산물), 단백질체는 게놈 발현의 최종 산물이다.
- Yang et al.은 이질적 소스의 질병 식별자를 UMLS(Unified Medical Language System) 코드로 매핑하여 통합 KG를 구축했다.
- 단백질-단백질 상호작용(PPI, Protein-Protein Interaction) 네트워크와 단백질-질병 연관으로 질병 경로(disease pathway)를 계산적으로 발견한다.

### 4.2.1 Creating a KG from PPI and Protein-Disease Networks

- 목표는 알려진 경로에서 출발해 질병과 연관된 잠재 단백질·경로를 예측하는 것으로, 결과 KG는 단일 분할(monopartite) PPI 네트워크와 이분(bipartite) 질병-단백질 네트워크로 구성된다.
- SNAP(Stanford Network Analytics Project) 데이터로 PPI 네트워크(21,559개 단백질, 342,354개 상호작용), DisGeNET의 단백질-질병 연관, Disease Ontology 기반 질병 분류, NIH 유전자 정보를 순차 import한다.
- Neo4j에서 NODE KEY 제약과 LOAD CSV ... IN TRANSACTIONS 패턴으로 데이터를 적재한다.

### 4.2.2 ~ 4.2.3 Analysis of the KG

- WCC(Weakly Connected Component)는 비연결 하위그래프를 찾는 커뮤니티 탐지로, GDS(Graph Data Science) 라이브러리로 실행하며 PPI가 27개 컴포넌트(하나는 21,521개 단백질로 거대)임을 보였다.
- Louvain 모듈성 알고리즘은 무작위 네트워크 대비 밀도로 모듈성 점수를 최대화하여 더 조밀하게 연결된 커뮤니티(48개, 모듈성 약 0.55)를 찾는다.
- 도메인 특화 분석: 질병 경로 Hd=(Vd,Ed)는 PPI 부분그래프이며, 최대 경로 컴포넌트 상대 크기, 밀도(density), 전도도(conductance) 세 지표로 특성화한다.
- 질병 경로는 PPI 네트워크 내에서 파편화(중앙값 16개 컴포넌트, 최대 컴포넌트에 중앙값 21% 단백질)되고 내부 밀도가 낮으나(중앙값 0.07) 외부 전도도는 높다(중앙값 0.96).

```
   Disease --ASSOCIATED_WITH--> Protein
                                   |
                              INTERACTS_WITH
                                   v
                                Protein   (PPI monopartite network)
   Disease pathway Hd = subgraph of PPI induced by disease's proteins
```

---

## 3. Pharmaceutical Applications of KGs

> 신약 개발은 약 14억 달러·15년이 들고 성공률이 낮아, 기존 약물 정보를 활용한 약물 분석·재창출(repurposing)이 기간·실패율·비용을 크게 줄인다.

- KG는 약물 상호작용 예측, 약물이 상호작용할 분자 표적 식별, 기존 약물로 치료 가능한 새 질병 결정에 활용된다.
- Himmelstein et al.의 Hetionet은 29개 공개 자원에서 화합물·질병·유전자·해부구조·경로 등을 연결한 이종 네트워크(47,031 노드 11종, 2,250,197 관계 24종)다.
- 메타패스(metapath)는 첫 타입과 마지막 타입 노드 사이의 잠재적 실제 경로를 기술하는 노드·관계 클래스 시퀀스로, 스키마를 "질의"하여 연결 패턴을 찾는다.
- 경로 수(PC, Path Count)는 각 경로를 1로 세지만, DWPC는 각 경로에 PDP(Path-Degree Product)를 부여하여 차수(degree)를 -w승(감쇠 지수 damping exponent)하여 곱해 "잘 알려진 노드" 편향을 줄인다.

### 4.3.1 ~ 4.3.2 Deep Analysis & LLM Interpretation

- 셀리악병(celiac disease) 사례에서 DWPC로 GO(Gene Ontology) 프로세스 농축(enrichment)을 분석해 'T cell costimulation'을 최상위로 식별(PC 정렬과 달리 DWPC는 880개 유전자 연결 프로세스를 하위로 밀어냄).
- GWAS(Genome-Wide Association Study) 기반 연관과 단백질 상호작용을 메타패스에 추가하면 당단백질(glycoprotein) 관련 프로세스 등 더 질병 특화된 결과가 나온다.
- LLM은 DWPC 정량 순위를 임상적으로 실행 가능한 통찰(예: T세포 공동자극과 셀리악병 자가면역성의 관계)로 종합·해석하는 지능형 해석자 역할을 한다.

---

## 4. Clinical Applications of KGs

> 임상 응용은 초기 단계로, 정밀의료(precision medicine)를 위해 오믹스 데이터와 EHR(Electronic Health Record)을 임상 의사결정에 통합하는 것이 장기 목표다.

- EHR은 해석이 어렵고 주관성이 크며 정보 누락이 있어, 임상 KG는 EHR과 다중오믹스·온톨로지를 병합한다(노드: 환자·약물·질병).
- 환자 여정 매핑(patient journey mapping)은 사람들이 의료 서비스에 진입·경험·이탈하는 방식을 이해하며, 표준 진료를 정의하는 임상 경로(clinical pathway)와 비교된다.
- 프라이버시 문제 대응: 실제 환자 데이터 대신 질병·증상·치료 결과의 통계 정보(가중치 포함)만 KG에 저장하거나, 비식별화·익명화된 일반 임상 KG를 구축한다.
- CKG(Clinical Knowledge Graph)는 33개 노드 레이블과 51개 관계 타입을 연결해 변경된 기능·약물 제안·교란 요인을 질의할 수 있게 한다.
- 사례 질의는 특정 단백질(ACTC1~P68032)이 여러 심근병증(cardiomyopathy)과 강하게 연관됨을 보여 임상시험 환자 계층화의 출발점을 제공한다.
- LLM은 이러한 단백질-질병 연관을 실용적 임상 의사결정으로 변환하나, 저자들은 의사·연구자의 적절한 해석 없이 LLM 결과를 사용하지 말 것을 강조한다(목표는 인간 대체가 아닌 IAS로의 강화).

---

## Summary (핵심 정리)

- 구조화 소스 기반 KG는 엔티티 해소(entity resolution)·스키마 정렬·데이터 품질 검증을 통한 체계적 통합이 필요하며, WCC·Louvain은 전역 구조와 커뮤니티 조직에 대한 통찰을 준다.
- 밀도·전도도·최대 연결 컴포넌트 상대 크기 같은 부분그래프 지표와 DWPC 같은 경로 기반 지표가 KG 품질·관계 패턴을 정량 평가한다.
- Hetionet·PPI·CKG가 통합·분석 기법의 테스트베드가 되며, LLM 보조 해석이 정량 지표를 도메인 특화 통찰·연구 가설로 변환한다.
