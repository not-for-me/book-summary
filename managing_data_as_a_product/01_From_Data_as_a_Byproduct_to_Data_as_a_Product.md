# 01. From Data as a Byproduct to Data as a Product

## 챕터 개요 (3줄 요약)

- 지난 30년간 모놀리식(monolithic) 데이터 플랫폼이 어떻게 진화해 왔는지(DWH, 데이터 레이크, MDS)와 그 한계를 설명한다.
- 플랫폼이 실패하는 근본 원인은 기술 선택이 아니라 모듈성(modularity) 부재로 인한 우발적 복잡성(accidental complexity)의 통제 불능임을 시스템 사고로 분석한다.
- 해결책으로 "데이터를 제품으로 관리(data as a product)"하는 패러다임 전환의 필요성을 제시한다.

---

## 1. Reviewing the history of monolithic data platforms

> 데이터 플랫폼의 실패는 대부분 잘못된 도구(tool) 선택이 아니라, 우발적 복잡성을 키우는 사회-기술적(socio-technical) 아키텍처 접근 방식 때문에 발생한다.

- 시스템 복잡성 = 컴포넌트 수 × 컴포넌트 간 상관관계의 수이며, 상관관계가 높으면 복잡성은 대략 2차 함수(quadratic)로 증가한다.
- 복잡성은 본질적 복잡성(essential complexity, 압축 불가)과 우발적 복잡성(accidental complexity, 통제 가능)으로 나뉜다.
- 현대 데이터 관리는 개별 기술 도구보다 아키텍처적 측면에 초점을 맞춘다.
- DWH(Data Warehouse): 80년대 말 IBM에서 제안된 중앙집중 저장소로, 단일 진실 공급원(single source of truth) 역할을 하며 생산자와 소비자를 분리한다.
- 데이터 레이크(Data Lake): 빅데이터와 ML(Machine Learning)/AI(Artificial Intelligence) 워크로드 대응을 위해 분산 저장·연산(HDFS, MapReduce) 기반으로 등장했으나 관리 부실 시 데이터 늪(data swamp)이 된다.
- MDS(Modern Data Stack): 클라우드 네이티브 포인트 솔루션의 생태계로, 도구 단편화가 운영 복잡성을 다시 키웠다.

### Data warehouse
- 통합 비용을 한 번만 지불해 여러 분석 사례에 재사용하지만, 새 데이터 소스 통합의 리드타임(lead time)이 길어지는 단점이 있다.

### Data lake
- 스키마 온 리드(schema on read)로 원시 데이터를 직접 질의할 수 있으나, 초기엔 비구조적 접근으로 인해 관리가 어려워졌다.
- 현대 데이터 레이크는 메달리온 아키텍처(medallion architecture, 계층형)로 민첩성과 통제 사이 균형을 잡는다.

### DWH versus data lake
- 두 접근은 서로의 한계를 보완하며 수렴했고, 오늘날 둘 중 하나만으로도 플랫폼 구축이 가능하나 많은 조직이 하이브리드를 선호한다.

---

## 2. Understanding why monolithic data platforms fail

> 데이터 플랫폼은 출시 전이 아니라 시간이 지나면서 복잡성의 무게에 짓눌려(complexity catastrophes) 진화·생존하지 못해 실패한다.

- 기술은 문제의 증상은 해결했지만 우발적 복잡성의 근본 원인은 해결하지 못했다.
- 근본 원인은 기술·조직 양면의 모놀리식 아키텍처이며, 핵심은 탈중앙화 부재가 아니라 모듈성(modularity)의 부재이다.
- 모듈은 명확한 인터페이스로 내부를 외부와 구분하고, 내부 응집도는 높고 다른 모듈과의 결합도(coupling)는 낮아야 한다.
- 분산되어 있어도 결합도가 지나치게 높으면 분산 모놀리스(distributed monolith)에 불과하다.
- 모듈화는 정보 은닉(information hiding)과 추상화(abstraction)로 인지 부하(cognitive load)를 줄여 복잡성을 통제한다.

### Failure loops
- 시스템 사고(systems thinking)에서 원인과 결과는 시간·공간적으로 분리되며, 피드백 루프(강화 루프 reinforcing, 균형 루프 balancing)로 분석한다.
- "부담 전가(shifting the burden)": 근본 치료 대신 새 기술로 증상만 임시 완화하다 같은 문제를 반복한다.
- "성장의 한계(limits to growth)": 데이터팀의 인지 부하가 한계에 달하면 기술 부채(technical debt)가 쌓이고 플랫폼이 레거시화한다.
- 해결책은 모듈화이나, 선행 비용과 성장 속도 둔화를 동반해 조직적 후원 확보가 어렵다.

```
            (R1) reinforcing
    births  ----------------->  population
       ^                            |
       |                            | (B1) balancing
       +----------------------------+
                                    v
                                  deaths
   [Causal Loop Diagram: population dynamics]
```

---

## 3. Exploring why we need to manage data as a product

> 표면적 증상이 아니라 빙산 모델(iceberg model)의 깊은 레버리지 포인트(leverage points), 특히 멘탈 모델부터 바꾸는 패러다임 전환이 필요하다.

- WEF(World Economic Forum)의 4차 산업혁명에서 데이터는 애플리케이션보다 더 오래 남는 핵심 자산이 된다.
- 개념적으로는 애플리케이션 중심에서 데이터 중심(data-centric)으로 이동했으나, 실무 멘탈 모델은 아직 전환되지 못했다.
- 데이터는 단순 축적으로 가치가 생기지 않으며 생애주기 전반의 지속적 관리가 필요하다(가치는 시간이 지나면 저하).
- 데이터 관리의 핵심 역량은 수집·처리·공유가 아니라 식별·기술(description)·표준화로 재정의되어야 한다.
- 데이터 관리는 데이터팀만의 일이 아니라 조직 전체(생산자=애플리케이션팀, 소비자=비즈니스)의 책임이다.

### Being data-centric
- "데이터는 우주의 중심이며 애플리케이션은 일시적이다"(The Data-Centric Manifesto) — 데이터를 조직 중심에 두는 멘탈 모델이 필요하다.

### Data product thinking
- 데이터를 제품으로 관리한다는 것은 제품 관리(product management) 실천을 데이터에 적용하는 것이다.
- 프로젝트는 산출물(output)·예산·기한으로 평가되나, 제품은 성과(outcome)·비즈니스 문제 해결로 평가되며 폐기될 때까지 동일 팀이 관리한다.
- 데이터 제품(data product)은 아키텍처의 모듈화 단위이며, 플랫폼은 비즈니스 도메인 기반 제품팀이 구성한다.

### Putting it all together
- 패러다임 전환은 멘탈 모델 → 조직 구조 → 운영 실천 순으로 모든 레벨에서 시너지를 이루며 개입해야 성공한다.

---

## Summary (핵심 정리)

- 모놀리식 플랫폼은 스스로 생성한 복잡성을 감당하지 못해 세대를 거듭하며 같은 실패를 반복해 왔다.
- 기술 혁신은 증상만 해결했을 뿐 근본 원인을 해결하지 못했으므로, 데이터를 제품으로 다루는 패러다임 전환이 필요하다.
- 이 전환은 멘탈 모델부터 운영 실천까지 조직 전반을 아우르는 시스템적 변화로, 플랫폼을 "성장 후 폐기"에서 "적응과 진화"로 이끈다.
