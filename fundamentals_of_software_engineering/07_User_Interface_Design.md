# 07. User Interface Design

## 챕터 개요 (3줄 요약)
- UI(User Interface)는 UX(User Experience)의 "무엇"이며 사용자에게 UI는 곧 애플리케이션 그 자체다.
- 사용성(usability), 접근성(accessibility), 지역화(L10N)/국제화(I18N)를 처음부터 고려하고 사용자를 깊이 이해해야 한다.
- 대비(Contrast)·반복(Repetition)·정렬(Alignment)·근접(Proximity)의 디자인 원칙과 명확한 오류 메시지, 파괴적 행동의 마찰 추가가 핵심이다.

---

## 1. Designing for Everyone
> 접근성, 사용성, 포용성은 서로 다르지만 연결된 개념으로 다양한 배경과 능력의 사람들이 제품을 쓸 수 있게 한다.

- a11y(accessibility), L10N(localization), I18N(internationalization) 같은 숫자약어(numeronym)로 자주 불린다.
- ADA(Americans with Disabilities Act), Section 508, EAA(European Accessibility Act) 등 법규의 적용을 받을 수 있다.
- 접근성·지역화·국제화는 처음부터 고려하는 것이 나중에 개조(retrofit)하는 것보다 훨씬 쉽다.
- 개발자 본인은 (개발자 도구가 아닌 한) 실제 사용자 집단을 대표하지 않는다.

### What Is Usability?
- 사용성은 "의도된 청중에게 애플리케이션이 얼마나 쓰기 쉬운가"로 귀결된다.
- 품질 요소: 학습성(learnability), 효율성(efficiency), 기억성(memorability), 발견성(discoverability), 오류 처리, 사용자 만족, 접근성.

### What Is Accessibility?
- 접근성은 다양한 능력의 사람이 소프트웨어를 쓸 수 있게 하는 것이다.
- 시각·청각·손재주(dexterity)·인지(cognitive) 과제를 고려한다.
- 접근성은 장애인뿐 아니라 모든 사용자를 돕는다(연석 경사로처럼 — curb cut effect).

### Localization & Internationalization
- I18N은 앱을 다양한 언어/문화에 적응 가능하게 설계하는 것(예: 레이블을 properties 파일에서 로드).
- L10N은 대상 시장의 언어·통화·날짜 형식·단위에 맞게 인터페이스를 적응시키는 것이다.
- 일부 언어는 RTL(Right-To-Left, 아랍어/히브리어)이고 레이블 확장(label expansion)도 고려해야 한다.

---

## 2. Know Your User
> 사용자와 그들이 인터페이스를 사용할 환경을 이해하지 않고는 사용성 높은 앱을 만들 수 없다.

- 사용자의 컴퓨터 경험, 기능적 제약, 기기 종류, 교육 수준, 사용 빈도, 사용 환경 등을 질문한다.
- "Don't Assume" — 사용자에 대한 가정은 강제되지 않은 오류로 이어진다(노인 컴퓨터 기증 일화).
- 시끄러운 환경이면 알림음 대신 대안 알림을, 매뉴얼을 안 읽는다고 가정하고 쉬운 길이 옳은 길이 되게 한다.

### Secondary Users & You Are Not Your User
- 1차 사용자가 다른 사람(2차 사용자)을 대신해 작업하는 경우가 많다(고객 서비스 상담원 예).
- 나쁜 사용성은 실제 비용을 발생시키고 시장 평판에 영향을 준다("11번 클릭" 일화).
- 사용자는 개발자가 하지 않을 행동을 한다(폰트 확대로 후보 이름이 가려져 여론조사가 무효화된 사례).

### The Tyranny of Defaults & Impact of Culture
- 기본값(default)의 힘을 과소평가하지 말고, 고객에게 묻거나 테스트하며, 파괴적 행동은 기본값을 "안전"하게 한다.
- 텔레메트리(telemetry)로 추측이 아닌 데이터에 반응해 인터페이스를 진화시킨다.
- 문화에 따라 멘탈 모델, 레이아웃 방향, 색상 함의(빨강=동양 행운/남아공 애도)가 다르다.

---

## 3. Maximizing Usability
> 모든 품질을 극대화하기 전에 "애플리케이션이 어떻게 사용될 것인가"라는 가장 중요한 질문을 먼저 생각한다.

- 매일 종일 쓰는 앱은 효율성·기억성·만족이 중요하고, 신규 가입 유도 앱은 학습성·발견성이 핵심이다.
- 효율성: 매일 쓰는 앱은 클릭 몇 번 절약이 큰 효과("작은 수 × 큰 수 = 큰 수").
- 기억성·학습성·발견성은 밀접하며 맥락이 가이드다(매일 쓰면 기억성↑, 산발적이면 학습성/발견성↑).

---

## 4. Principles of Design
> Robin Williams의 대비·반복·정렬·근접 원칙은 문서뿐 아니라 인터페이스 디자인에도 동일하게 적용된다.

### Contrast
- 대비는 가장 효과적인 도구이며 두 요소가 다르면(색·폰트 크기·선 굵기) 대비가 생긴다.
- 효과적이려면 정말 다르게 — "Don't be a wimp", 회색 두 음영보다 빨강과 검정처럼 뚜렷하게.

### Repetition
- 핵심 시각 요소(폰트·색·레이아웃·불릿)를 반복해 친숙함과 응집성을 준다.
- 같은 사이트에 있음을 알려주지만, 색 등 과용은 금물.

### Alignment
- 가장 쉬운 원칙으로 모든 요소를 신중히 배치하고, 좌/우 정렬로 날카로운 수직선을 만든다.
- 가운데 정렬은 들쭉날쭉해 가독성이 떨어지므로 피하고, 한 정렬 방식을 일관되게 반복한다.

### Proximity
- 가까이 묶인 항목은 관련 있는 것으로 인식되고, 관련 없는 것은 떨어뜨린다.
- 시각 그룹 사이에 여백(whitespace)을 두고, 그룹은 5개 이하로 최소화한다.

```
Design Principles (C-R-A-P):
  Contrast   -> make different things VERY different
  Repetition -> repeat visual elements for cohesion
  Alignment  -> align everything; prefer left/right edges
  Proximity  -> group related items; whitespace between groups
```

---

## 5. Make the Right Thing the Obvious Thing
> 발견성은 학습의 핵심이며, 더 나은 대안이 없는 한 명백한 접근을 택한다.

- 기능이 예상 밖에 숨겨져 있으면 학습에 훨씬 오래 걸린다(오디오 녹음 다이얼로그 예).
- 발견성은 기억성에 기여하므로 의심스러우면 더 명백한 접근을 택한다.
- 핀치 투 줌(pinch-to-zoom)처럼 발견성이 낮아도 학습성·기억성이 뛰어나면 예외가 된다.
- 키보드 단축키는 효율적이나 기억성이 낮을 수 있어 메뉴에 함께 표시(IntelliJ Find Action)한다.
- 올바른 필드 타입, 힌트, 필드 마스크, 시각 단서로 사용자가 잘못하는 것을 방지한다.

### Good Error Messages
- 좋은 오류 메시지는 무엇이 잘못됐는지 이해시키고 대안을 제시한다(Gmail의 "Try now" 예).
- undo/revert를 지원해 사용자가 복구하며 탐색할 수 있게 한다.

### Destructive Actions
- 파괴적·되돌릴 수 없는 행동(계정/리포지토리 삭제)에는 마찰(friction)을 추가하는 것이 옳다.
- GitHub 리포 삭제 예: Danger Zone을 하단 배치(proximity), 빨강 대비(contrast), 단계별 확인, 이름 직접 입력.

---

## Summary (핵심 정리)
- 사용성을 가장 먼저 떠올리지 않더라도 최종 사용자를 결코 놓치지 말아야 하며, 나쁜 사용성의 비용을 과소평가하지 않는다.
- 대비·반복·정렬·근접 원칙은 단순하지만 평범한 앱과 최고의 사용자 경험을 가르는 차이를 만든다.
- 좋은 UI는 접근성·지역화·국제화를 고려해 영어 사용자나 완벽한 시력의 사용자만이 아닌 모두가 쓸 수 있게 한다.