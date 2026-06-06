# 20. Static Analysis

## 챕터 개요 (3줄 요약)
- 정적분석은 프로그램을 실행하지 않고 소스코드를 분석해 버그·안티패턴을 찾고, 베스트프랙티스 codify·API 현행화·기술부채 감소까지 수행한다.
- 효과적 정적분석의 두 축은 확장성(scalability)과 사용성(usability)이며, 핵심 교훈은 개발자 행복·핵심 워크플로 통합·사용자 기여 권한 부여다.
- AI 시대에 코드가 폭증할수록, 낮은 false-positive로 신뢰를 쌓고 코드리뷰에 통합된 정적분석 생태계를 구축하는 것이 품질의 시니어 레버리지가 된다.

---

## 1. Characteristics of Effective Static Analysis (효과적 정적분석의 특성)
> 분석 기법 자체보다 확장성·사용성에 초점을 맞춘 것은 비교적 최근의 발전이다.

- Scalability: 수십억 줄 코드베이스 대응 — 샤딩·증분 분석, 변경된 파일/라인에만 결과 표시; 분석 종류도 사내 기여로 확장.
- Usability: 비용-편익 트레이드오프 — 잘 돌아가는 코드를 "고치다" 버그 유발 위험 → 새로 도입된 경고(또는 수정된 라인)에 집중.
- 개발자 시간은 귀하다 — 자동 수정 가능한 것은 자동 수정, 실제 코드 품질에 부정적인 이슈만 표시; 매끄러운 워크플로 통합.

---

## 2. Key Lessons (핵심 교훈)
> Google이 배운 세 가지 교훈.

### Focus on Developer Happiness (개발자 행복)
- 측정하지 않으면 고칠 수 없다 — 낮은 false-positive 도구만 배포, 실시간 피드백 루프로 신뢰 구축.
- "Effective false positive"(체감 false-positive): 개발자가 보고도 긍정적 행동을 취하지 않으면 false positive — 기술적으로 맞아도 메시지가 혼란스럽거나 사소하면 동일 취급.

### Make Static Analysis Part of Core Workflow (핵심 워크플로 통합)
- 코드리뷰 통합이 sweet spot — 변경 마인드셋, 리뷰 대기 시간에 분석 실행, 리뷰어의 peer pressure, 리뷰어 시간 절약.

### Empower Users to Contribute (사용자 기여 권한)
- 도메인 전문가가 새 분석/체크 작성 — Refaster(전/후 코드 스니펫으로 분석기 작성) 같은 단순 API로 누구나 기여.

---

## 3. Tricorder: Google's Static Analysis Platform (트라이코더)
> Tricorder의 핵심 차별점은 "가치 있는 결과만 전달"하는 데 대한 집요한 집중이다.

```
Tricorder check criteria (new check)
1. Understandable - any engineer can understand output
2. Actionable & easy to fix - include guidance/fix
3. < 10% effective false positives (>=90% real issues)
4. Significant impact on code quality
=> 100+ analyzers, 30+ languages, ~5% effective FP rate, 50k+ changes/day
```

- 마이크로서비스 아키텍처 — Critique diff 뷰에 회색 코멘트로 경고 표시.
- 통합 도구: Error Prone(Java)·clang-tidy(C++)(AST 안티패턴), Deleted Artifact Analyzer, IfThisThenThat, Chrome Finch(A/B 실험), 바이너리 크기 체커 — 대부분 intraprocedural.
- 피드백 채널: "Not useful" 버튼(분석기 작성자에게 버그 제출), "Please fix" 버튼; high "Not useful" 비율 분석기는 비활성화 — 메시지 텍스트 개선만으로도 해결되기도(%s printf 사례).
- Suggested Fixes: 자동 수정 제공(Critique/CLI에서 적용) — 스타일 이슈는 자동 수정(포매터); "Please Fix" 수천 번/일, 자동 수정 ~3000번/일.

### Per-Project Customization (프로젝트별 커스터마이징)
- 신뢰 구축 후 선택적(optional) 분석기 추가(Proto Best Practices 등) — optional → 피드백 개선 → on-by-default 졸업.
- 핵심 통찰: 사용자별이 아닌 "프로젝트별" 커스터마이징 — 팀 전원이 일관된 결과; 사용자 커스터마이징은 버그를 숨기고 피드백을 억제(린터 사례).

---

## 4. Other Integration Points (다른 통합 지점)
> 코드리뷰 외에도 여러 지점에서 통합한다.

- Presubmits: 커밋 차단 체크("DO NOT SUBMIT" 검사, 테스트 동반, 포매팅) — 팀별 커스텀 presubmit으로 더 엄격한 기준 가능(LSC 시 일부 skip).
- Compiler Integration: 가능하면 컴파일러로 — 무시 불가한 경고; Error Prone "ERROR" 체크는 actionable·effective FP 없음·correctness만; 신규 체크 도입 전 MapReduce로 전체 코드베이스 정리 후 활성화.
  - 컴파일러 경고는 절대 표시 안 함(무시되므로) — 빌드 깨는 에러로 만들거나 숨김(Go는 미사용 변수도 에러).
- IDE: 빠른 분석(<1초)이 필요하고 IDE마다 일관성·인기 변동 문제로 리뷰보다 지저분; 코드 브라우징 시 전체 결과 보기(보안 분석·클린업 계획)는 유용.

---

## Summary (핵심 정리)
- 개발자 행복에 집중하라 — 분석 사용자와 작성자 간 피드백 채널을 구축하고 false-positive를 공격적으로 줄여라.
- 정적분석을 핵심 개발 워크플로의 일부로(주로 코드리뷰, 추가로 컴파일러·presubmit·IDE·코드 브라우징).
- 사용자가 기여하도록 권한을 부여하라 — 도메인 전문가의 분석 기여로 확장 — AI 시대 코드 폭증 속 품질의 시니어 레버리지.
