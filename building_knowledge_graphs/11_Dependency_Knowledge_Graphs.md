# 11. Dependency Knowledge Graphs

## 챕터 개요 (3줄 요약)
- 의존성 모델링은 프로젝트 태스크, 소프트웨어 취약점 전파, 기업 지분 체인(UBO, 궁극적 수익소유자) 등 다양한 문제의 공통 본질이며, 지식그래프가 자연스러운 표현 수단이다.
- 관계형 SQL은 재귀 경로 분석에 부적합하나, Cypher의 가변 길이 패턴 `-[:DEPENDS_ON*]->`은 임의 깊이를 직관적·컴팩트하게 처리한다.
- 핵심 모델: 가중·시간유효성 의존성, 두 가지 다중의존성(가산/aggregate vs 중복/redundant), 영향 전파(impact propagation), 그래프 검증, SPOF·근본원인 분석(RCA).

## Dependencies as a Graph
> 표 형식 의존성은 "A와 F 사이 숨은 의존성이 있나?" 같은 질의에 다중 JOIN이 필요하나, 그래프에선 가변 길이 패턴 한 줄로 해결된다.

- 각 직접 의존성 = 방향 관계, 이행적(transitive) 의존성 네트워크 = 그래프.
- DAG(Directed Acyclic Graph)가 정상 — 깊이에 무관하게 동일 쿼리로 동작(유지보수 비용 ↓).

## Advanced Dependency Modeling
> 속성 그래프는 관계 타입+속성으로 "정성적(qualified) 의존성"(강도·시간유효성)을 자연스럽게 표현한다.

- Qualified: 지분율(OWNS.percentage), 용량(CONSUMES.capacity) 등 합성 가능한 수치를 관계 속성으로.
- Temporal validity: 관계에 start/end 속성 → 시점 t 파라미터로 활성 의존성만 탐색.
- 다중의존성 2종:
  - **Aggregate(가산)**: 동시 활성, 가중 합. `parent_impact = Σ(child_impact × child_weight)`. 예: 포트폴리오 자산 비중, 본딩된 통신 링크 용량.
  - **Redundant(중복/보호)**: 하나만 활성, 나머지 예비. `parent_impact = min(child_impacts)`. 임계(threshold) 일반화 가능. 예: HA 서버 쿼럼, 공급망 대체 경로.

```
// 가변 길이 의존성 경로
MATCH path = (:Element {id:'A'})-[:DEPENDS_ON*]->(:Element {id:'F'})
RETURN path
// Aggregate 영향 전파
parent_impact = child1_impact*w1 + child2_impact*w2 + ...
// Redundant 영향 전파
parent_impact = min(child1_impact, child2_impact, ...)
```

## Impact Propagation & Validation
> 영향 분석은 (1) 위상적 최대 영향 범위 → (2) 의미(가중·다중)를 반영한 상세 평가 순으로 한다.

- 최대 영향: `(e)-[:DEPENDS_ON*]->(impacted)` 로 영향 가능 노드 집합 산출.
- 검증 4종: (1) 사이클 없음(DAG) — `(e)-[:DEPENDS_ON*]->(e)` 탐지, (2) Aggregate 가중합 = 100%(또는 total), (3) 소비 ≤ 생산(producer/consumer 균형), (4) Redundant 멤버 수 ≥ threshold.
- 실시간 이벤트 스트림 보강에 활용 — 대량 이벤트의 필터링·우선순위화.

## SPOF & Root Cause Analysis
> SPOF(단일 장애점)는 "여러 의존 체인이 한 노드로 수렴"하는 패턴, RCA는 영향 전파의 역방향이다.

- SPOF: `(spof)<-[:DEPENDS_ON*]-(e)-[:DEPENDS_ON*]->(spof)` — 표면적 이중화 뒤에 숨은 공통 의존(예: 두 서버가 같은 VM) 적발.
- RCA(근본원인): 증상(symptoms)에서 leaf 노드(다른 것에 의존 안 함)를 후보로 군집화 → precision·recall·F-score로 순위. 가장 균형 잡힌 후보가 진짜 원인일 가능성 높음.
- 사례 Vanguard Group(대형 자산운용): 400만 줄 모놀리스 Java를 마이크로서비스로 전환 시 코드 의존성 그래프로 가시화·dead code 제거·서비스 호출 수 제약.

> [모델링 관점 - 주식시장 도메인 적용]
> 이 장은 주식시장 모델링에 가장 직접적으로 적용되는 핵심 장이다. 책 도입부에서 명시된 예시 자체가 "기업 지분 체인을 따라 UBO(궁극적 수익소유자)를 추적하는 것은 자금세탁방지(AML)·테러자금조달방지 규제의 핵심"이다. 적용: (1) 지분 그래프 = (Company)-[:OWNS {percentage}]->(Company) 가산 다중의존성으로 모델링 → 가변 길이 경로로 간접 지분율(예: A가 B 60%, B가 C 50% → A의 C 실효 지분 30%)을 `Σ(weight 곱)`로 계산해 UBO 식별. (2) 시스템 리스크 전파 = 한 기업 디폴트의 충격이 지분·공급·채권 관계를 따라 어디까지 `parent_impact` 전파되는지 정량화. (3) Redundant(min) vs Aggregate(가중합) 구분이 중요 — 공급처 다변화(redundant, 하나 살아있으면 OK)와 매출 비중(aggregate, 비중만큼 타격)을 다르게 모델링. (4) SPOF 탐지 = 표면상 분산된 포트폴리오가 실은 동일 원자재/금리/단일 거래상대방에 수렴하는 "숨은 집중 리스크" 적발 — 분산투자의 환상을 깨는 강력한 도구. (5) RCA = 동시다발 시장 이상의 공통 원인(특정 섹터/매크로 충격) 역추적. 검증 로직(지분합 100%, 사이클 없음=순환출자 탐지)도 한국 시장의 순환출자 규제 점검에 그대로 쓰인다.

## Summary (핵심 정리)
- 의존성 지식그래프는 가변 길이 패턴으로 임의 깊이 전파를 직관적·효율적으로 처리하며, 가중·시간·다중(가산/중복) 의존성을 속성 그래프로 자연스럽게 표현한다.
- 영향 전파(aggregate=가중합, redundant=min), 그래프 검증(사이클·합계·균형·임계), SPOF·RCA가 핵심 분석 도구다.
- 주식시장에서는 지분 체인 UBO 추적·시스템 리스크 전파·숨은 집중 리스크(SPOF)·순환출자 탐지에 직접 적용되며, redundant/aggregate 구분과 가중치 합성이 정확한 리스크 모델링의 관건이다.
