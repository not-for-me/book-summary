# 11. Powering Up Your Productivity

## 챕터 개요 (3줄 요약)
- 생산성 높은 엔지니어는 자신의 도구(에디터, 명령줄, 유틸리티)를 통제하고 자기에게 맞게 커스터마이즈한다.
- 키보드 단축키와 명령줄(CLI)을 마스터하면 흐름 상태(flow)를 유지하며 작업을 가속하는 force multiplier가 된다.
- 매주 한 가지씩 배우고, 반복 작업을 자동화하며, 개인 지식 관리(PKM)로 정보를 축적하는 습관이 핵심이다.

---

## 1. Optimizing Your Development Environment
> 도구 설치는 시작일 뿐이며, 자신의 필요에 맞게 개발 환경을 커스터마이즈해야 한다.

### Know Your Development Tools
- 셰프가 자기 칼을, 정비공이 자기 공구를 챙기듯 엔지니어는 자기 도구를 소유해야 한다.
- 목표는 깊은 집중의 흐름 상태(flow state, Csikszentmihalyi)이며, 명령을 찾느라 멈추면 집중이 깨진다.
- 매일 쓰는 기능은 손바닥처럼 알아야 하고, 매주 새 기능 하나씩 배운다(예: IntelliJ의 'tip of the day').
- 단, 커스터마이징은 시간 낭비가 될 수 있으니 타임박싱하고, 과하면 페어링이 어려워진다.

### Build Your Own Lightsaber
- 차의 시트/거울을 맞추듯 환경을 자신에게 맞춘다 — "자신만의 라이트세이버를 만들어라".
- 모노스페이스 폰트, 테마(다크 모드), OS 설정, 무료/저가 유틸리티로 마찰을 제거한다.
- 물리적 환경(의자, 모니터 높이, 스탠딩 데스크)도 조정하고, 더 나은 키보드(에르고독스 등)를 산다.
- 눈의 피로(eye strain)에 주의하고 휴식하며, 안경 처방 시 화면 사용을 알린다.

### Leverage the Power of the Command Line
- CLI(Command-Line Interface)는 처음엔 위협적이지만 곧 슈퍼파워이자 force multiplier가 된다.
- man 명령으로 매뉴얼을 찾고, cat/cut/grep/pbcopy를 파이프(|)로 연결해 복잡한 출력을 만든다.
- bat, fx, exa 같은 현대적 CLI 도구와 셰프 별칭(alias), 히스토리(Ctrl-R 역검색)를 활용한다.
- Oh My Zsh 같은 셸로 플러그인·테마·헬퍼를 더한다.

```
Pipe example (copy Git repo URL):
  cat .git/config | grep url | cut -f2 -d= | pbcopy
```

### Harness the Power of Keyboard Shortcuts
- 손을 홈 로(home row)에 유지하는 것이 목표 — 단축키는 빠르고 손목 부담을 줄인다.
- 터치 타이핑(touch typing)을 반드시 배운다(게임화된 온라인 도구 활용).
- OS와 에디터 단축키를 모두 배우고, Emacs 키 바인딩(Ctrl-F/B, Ctrl-A/E)으로 텍스트 탐색을 가속한다.
- Pastebot(클립보드 히스토리), TextExpander(텍스트 확장) 같은 도구로 작은 절약을 누적한다.

---

## 2. Strategic Automation
> 다른 사람을 위해 코드를 쓰듯, 자신을 돕는 코드도 두려워하지 말고 작성하라.

- 컴퓨터 과학에는 세 숫자만 있다: 0번, 1번, 그리고 n번 — 한 번 넘게 하면 무수히 한다고 보고 자동화를 고려한다.
- 정규표현식(regex)을 조금만 배워도 큰 도움이 되며, 30줄의 Python이 수 시간을 절약할 수 있다.
- 단, 자동화는 시간 낭비가 될 수 있으니 몇 번 해본 뒤 타임박싱하여 진행한다.
- AI에게 스크립트를 맡길 수도 있지만 반드시 읽고 테스트한다.

---

## 3. The Perpetual Pursuit of Productive Habits
> 단축키와 유틸리티는 끝이 없으므로 한 달에 다 익히려 하지 말고 매일 조금씩 개선한다.

### Collaborative Learning
- 동료가 최고의 팁/트릭 원천이며, 모르는 도구를 보면 묻고, 어렵게 하는 동료에겐 쉬운 길을 보여준다.
- 개념 강화를 위해 3~4번 반복하고, 런치앤런(lunch-and-learn) 세션으로 공유한다.
- "see one, do one, teach one" 접근과 언어 릴리스 노트 학습으로 진화에 발맞춘다.

### Personal Knowledge Management
- 정보 홍수 속에서 PKM(Personal Knowledge Management) 전략으로 흥미로운 정보를 저장한다.
- 메모하라 — 두뇌에서 외부 저장소로 오프로드(external brain)한다.
- 노트북 메모, Field Notes/Moleskine 노트, Org Mode/Evernote/Notion 등 자신에게 맞는 것을 쓴다.
- Notion은 학습·문서화·체크리스트·노트 테이킹의 "두 번째 뇌"로 활용 가능하다.
- 컨퍼런스 영상을 플레이리스트로 큐잉하고, 도구는 한 기기에 종속되지 않고 모바일에서도 동작해야 한다.

---

## Summary (핵심 정리)
- 좋은 엔지니어는 새 차의 시트를 맞추듯 자신의 도구킷을 통제하고 에디터·OS를 자기 것으로 만든다.
- 키보드 단축키와 명령줄은 force multiplier이며, 무료/저가 유틸리티가 하루를 크게 바꿀 수 있다.
- 암기에 의존하지 말고 지식 관리 습관을 기르며, 동료에게서 배우고 동료를 가르친다.