# 06. Leading at Scale

## 챕터 개요 (3줄 요약)
- 팀의 팀(team of teams)을 이끌면 "깊이(deep)"보다 "넓이(broad)"로, 기술 세부보다 고수준 전략과 사람 조직화로 역할이 이동한다.
- 규모의 리더십은 "세 가지 Always": Always Be Deciding(트레이드오프 결정·반복), Always Be Leaving(자가구동 조직 구축), Always Be Scaling(시간·주의·에너지 보호)로 요약된다.
- AI 시대에 개별 구현 비용이 낮아질수록, 모호한 문제를 조직이 스스로 풀게 만드는 리더의 메타전략과 자기 확장 능력이 핵심 레버리지가 된다.

---

## 1. Always Be Deciding (항상 결정하라)
> 중요하고 모호한(ambiguous) 문제에는 은탄환(silver bullet)이 없으며, 리더의 일은 그 순간 최선의 트레이드오프를 찾고 반복하는 것이다.

- 3단계 프로세스: 블라인더(blinders) 식별(현상유지 가정에 빠진 시각), 핵심 트레이드오프 식별, 결정 후 반복(decide, then iterate).
- "이건 원래 이렇게 해왔어"라는 코핑 메커니즘을 신선한 눈으로 의심하라.
- 완벽한 해를 찾는 "분석 마비(analysis paralysis)"를 피하려면 "이번 달엔 이걸 시도하고 다음 달에 재조정"이라며 반복을 편하게 만들어라.

### Case Study: Web Search Latency
> 품질(Quality) 추구가 지연(Latency)을 야금야금 늘리는 "공유지의 오염" 문제를, 지연을 1등 시민(first-class goal)으로 격상해 해결했다.

```
The Tension Triangle (pick two!)
          Good (Quality)
            /        \
           /          \
   Fast (Latency)---Cheap (Capacity/serving)
- improve one trait -> usually harms one of the other two
```

- 데이터 과학자가 지연이 사용자 참여(engagement)에 미치는 영향을 측정해, 품질의 단기 이득 vs 지연의 장기 손해를 정량적으로 비교하는 지표를 만들었다.

---

## 2. Always Be Leaving (항상 떠날 준비를 하라)
> 당신의 일은 문제를 푸는 것이 아니라, 당신 없이도 조직이 스스로 풀게 만드는 것이다(SPOF가 되지 말 것).

- Bus Factor 테스트: 일주일 휴가에도 메일을 계속 확인한다면 당신은 단일 장애점(SPOF)이다.
- 자가구동(self-driving) 팀 구축 3요소: 문제 공간 분할, 하위 문제를 리더에게 위임, 필요 시 조정·반복.
- 위임(delegation)은 자가구동 조직을 만드는 핵심 교육 수단 — "내가 정말 이걸 할 수 있는 유일한 사람인가?"를 물어라.
- 매일 물을 질문: "팀에서 나만 할 수 있는 일은 무엇인가?" (조직 정치 차단, 문화 조성, 고수준 전략 = 숲을 보기).
- 95% 관찰·경청, 5% 정확한 위치의 결정적 조정(분필 X 표시 우화 — "어디에 표시할지 아는 값 $9,999").

### Take care in anchoring team identity (팀 정체성)
- 팀을 특정 "제품(해결책)"이 아니라 일반 "문제"에 묶어라 — "우리는 Git 저장소 관리 팀"보다 "우리는 회사에 버전관리를 제공하는 팀"이 진화 가능하다.

---

## 3. Always Be Scaling (항상 확장하라)
> 가장 귀한 자원은 한정된 시간·주의·에너지이며, 이를 방어적으로 보호해야 확장이 지속된다.

### The Cycle/Spiral of Success (성공의 나선)
- 분석(Analysis) -> 고투(Struggle) -> 견인(Traction) -> 보상(Reward: 더 많은 일과 책임!).
- 압축 단계(compression): 같은 인력으로 기존 문제를 절반 자원·시간에 관리하며 새 문제를 떠안음 — Larry Page 표현 "uncomfortably exciting".

### Important vs Urgent (중요 vs 긴급)
> 리더가 되면 일이 반응적(reactive) 소방수로 변하기 쉽다 — Eisenhower: "긴급한 것은 중요하지 않고, 중요한 것은 결코 긴급하지 않다".

- 기법: 위임(Delegate), 중요-비긴급 작업에 전용 시간 블록 확보, 자신에게 맞는 추적 시스템 찾기(GTD 등).
- Learn to Drop Balls: Marie Kondo식으로 상위 20%(나만 할 수 있는 critical)만 식별해 집중하고 나머지 80%는 의도적으로 떨어뜨려라 — 중요한 것은 결국 돌아온다.

### Protecting Your Energy (에너지 보호)
- 진짜 휴가(최소 3일~1주, 완전 단절), 단절을 쉽게(업무 프로필 끄기), 진짜 주말, 일과 중 90분 주기 휴식, 멘탈 헬스 데이.

---

## Summary (핵심 정리)
- Always Be Deciding: 모호한 문제는 그 순간의 트레이드오프를 찾아 반복하는 것이다.
- Always Be Leaving: 당신 없이도 한 부류의 모호한 문제를 자동으로 푸는 조직을 만들어라.
- Always Be Scaling: 성공은 더 많은 책임을 낳으므로, 시간·주의·에너지를 능동적으로 보호하라 — AI 시대 시니어의 핵심 레버리지.
