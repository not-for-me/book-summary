# 03. Knowledge Sharing

## 챕터 개요 (3줄 요약)
- 조직의 성공은 전문성을 키우고 분산하는 데 달려 있으며, 그 토대는 모르는 것을 인정해도 안전한 심리적 안전(psychological safety)이다.
- 지식 공유는 질문하기·기록하기 같은 단순한 것부터 멘토십·문서·정규 교육·정적분석·표준화된 멘토십(Readability)까지 다층적 메커니즘으로 확장된다.
- AI 시대에 지식이 가장 중요한 무형 자본인 만큼, 부족(tribal) 지식과 문서 지식을 상호보완적으로 설계해 조직을 변화에 강하게 만드는 능력이 핵심이다.

---

## 1. Challenges to Learning (학습의 장애물)
> 강한 학습 문화가 없으면 조직 확장 과정에서 여러 안티패턴이 나타난다.

- Lack of psychological safety: 실수·위험 감수를 두려워하는 공포 문화.
- Information islands: 정보 단편화(fragmentation)·중복(duplication)·왜곡(skew)을 낳는 지식 섬.
- Single Point of Failure(SPOF): 한 사람에게만 핵심 정보가 있는 병목(Bus Factor와 연결).
- All-or-nothing expertise: 전문가와 초보만 있고 중간이 없는 양극화.
- Parroting(맹목적 모방)과 Haunted graveyards(두려움으로 건드리지 않는 코드 영역).

---

## 2. Philosophy & Psychological Safety (철학과 심리적 안전)
> 모든 전문가는 한때 초보였으며, 부족 지식과 문서 지식은 상호보완적이다.

- 문서 지식은 조직 전체로 확장되지만 일반적이고 유지비용이 든다; 인간 전문가는 맥락에 맞게 종합·판단한다.
- 심리적 안전은 효과적 팀의 가장 중요한 요소(Google 연구) — 배우려면 모름을 인정할 수 있어야 한다.
- Mentorship: Noogler(신입)에게 팀 외부 멘토를 배정해 "너무 많은 시간을 뺏는 것 아닌가" 걱정 없이 질문하게 한다.
- 대규모 그룹에서는 협력적(cooperative) 상호작용이 필수 — Recurse Center 사회 규칙: 가짜 놀람 금지, "well-actually" 금지, 백시트 드라이빙 금지, 미묘한 차별 금지.

---

## 3. Growing & Scaling Your Knowledge (지식 키우고 확장하기)
> 항상 배우고 항상 질문하라; 그리고 배운 것을 기록해 커뮤니티와 공유하라.

- Ask Questions: Noogler 램프업은 약 6개월; "모른다"고 말하는 것을 두려움이 아닌 기회로 받아들여라(리더가 모델링해야 함).
- Understand Context: "Chesterton's Fence" — 무언가를 바꾸기 전에 왜 거기 있는지부터 이해하라.
- 커뮤니티로 확장: Group Chats(빠른 Q&A), Mailing Lists(맥락 많은 질문·검색 가능 아카이브), YAQS(Stack Overflow형 Q&A 플랫폼).
- 가르치기: Office Hours, Tech Talks/Classes(g2g, engEDU), Documentation, Code(코드 자체가 지식; 코드리뷰는 양방향 학습 기회).

---

## 4. Scaling Organization's Knowledge (조직 지식 확장)
> 문화와 인센티브, 그리고 정규 정보원(canonical sources)이 조직 차원의 지식 확장을 가능하게 한다.

- Respect: "brilliant jerk(똑똑한 또라이)" 용인은 해롭다 — "Jerks are not good leaders".
- 인센티브와 인정: Peer bonus·kudos 같은 상향식(bottom-up) 인정과 직무 사다리(job ladder)의 명시적 기대치.
- Canonical sources: developer guides, go/ links(내부 URL 단축기·영구링크), codelabs(실습형 튜토리얼), static analysis(프로그래밍적으로 베스트프랙티스 전파).
- Staying in the loop: newsletters(Testing on the Toilet 등), communities(Google Groups).

```
Knowledge Sharing Mechanisms (low -> high structure)
Ask question / Write it down
   -> Group chat -> Mailing list -> YAQS
   -> Office hours -> Tech talks / Classes / Codelabs
   -> Documentation (g3doc, go/ links)
   -> Static analysis (automated best practices)
   -> Readability (standardized mentorship via code review)
```

---

## 5. Readability: Standardized Mentorship (표준화된 멘토십)
> Readability는 코드리뷰를 통해 언어별 베스트프랙티스를 회사 전체에 전파하는 표준화된 멘토십 프로세스다.

- 모든 CL(Changelist)은 readability 승인이 필요하며, 인증된 저자는 자기 CL에 암묵적 승인을 가진다.
- 중앙집중식 리뷰는 선형 확장이라는 비용을 치르지만, 모노레포(monorepo) 전반의 일관성과 정보 섬 방지라는 이득을 준다.
- EPR(Engineering Productivity Research) 연구: readability 보유 저자의 CL이 통계적으로 더 빨리 리뷰·제출되며, 학습·행동 변화로 이어진다.
- 단기 리뷰 지연·선행 비용을 감수하고 장기적 코드 품질·일관성·전문성을 얻는 의도적 트레이드오프.

---

## Summary (핵심 정리)
- 지식은 조직의 가장 중요한 무형 자본이며, 심리적 안전이 지식 공유 문화의 토대다.
- 질문하기·기록하기에서 시작해 인간 전문가와 문서·정적분석·Readability를 상호보완적으로 결합해 확장하라.
- AI 시대에 부족 지식과 정규 정보원을 설계하고 인센티브로 학습 문화를 키우는 것이 조직 회복탄력성의 핵심이다.
