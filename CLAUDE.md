# Books Summary — 작업 규칙

기술 서적을 챕터별로 한국어 요약한 모음입니다. 각 책은 하나의 폴더(snake_case)이고,
챕터는 `NN_Chapter_Title.md` 파일로 저장합니다.

## ⚠️ 필수: 책/챕터를 추가·수정하면 README.md도 함께 갱신한다

새 책 폴더를 추가하거나, 기존 책에 챕터 파일을 추가/삭제/이름변경할 때는
**반드시 같은 커밋에서 `README.md`를 동일하게 반영**한다. README는 전체 목록의
단일 출처(SSOT)이므로 폴더 구조와 항상 일치해야 한다.

### 새 책을 추가할 때

`README.md` 최하단(`## 목록`의 마지막 섹션 뒤)에 아래 포맷으로 섹션을 추가한다.
섹션은 책을 추가한 순서대로 아래에 append 한다.

```markdown
### <emoji> [<책 제목>](./<폴더명>)
<한 줄 한국어 소개 — 책의 핵심 주제>
📖 책: [<정식 책 제목> (Amazon)](<Amazon URL>)

<details>
<summary>챕터 목록 (<개수>)</summary>

- [01. <Chapter Title>](03_Resources/AI%20&%20Data%20&%20Engineering/Books/<폴더명>/<파일명>.md)
- [02. ...](...)

</details>
```

규칙:
- **폴더 링크**는 `./<폴더명>` (상대 경로), **챕터 링크**는
  `03_Resources/AI%20&%20Data%20&%20Engineering/Books/<폴더명>/<파일명>` 형태를 쓴다
  (기존 항목과 동일한 절대 스타일 유지). 경로의 공백은 `%20`으로 인코딩한다.
- 챕터 순서와 개수는 실제 폴더의 `.md` 파일과 정확히 일치시킨다.
- Amazon 링크는 **추측하지 말고** 실제 책을 확인해서 정확한 URL을 넣는다.
- emoji는 주제에 맞게 고르되 기존 섹션과 겹쳐도 무방하다.

### 기존 책에 챕터를 추가/변경할 때

해당 책 섹션의 `<details>` 안 챕터 목록과 `(<개수>)` 값을 실제 파일과 맞춘다.

### 검증

작업 후 아래로 폴더와 README의 불일치를 확인한다 (누락/stale 링크가 없어야 한다):

```bash
python3 - <<'PY'
import os, re, urllib.parse
readme = open("README.md", encoding="utf-8").read()
linked = set()
for m in re.finditer(r'\((?:\./)?(?:03_Resources/AI%20&%20Data%20&%20Engineering/Books/)?([^)]+\.md)\)', readme):
    linked.add(urllib.parse.unquote(m.group(1)).replace("03_Resources/AI & Data & Engineering/Books/", ""))
for d in sorted(os.listdir(".")):
    if os.path.isdir(d) and not d.startswith('.'):
        for f in sorted(os.listdir(d)):
            if f.endswith(".md") and f"{d}/{f}" not in linked:
                print("README 누락:", d, f)
for p in sorted(linked):
    if "/" in p and not os.path.exists(p):
        print("stale 링크:", p)
PY
```

## ⚠️ 필수: public 미러에도 함께 반영한다

이 저장소는 공개용 repo `public-knowledge-archive`의
`resource/AI & Data & Engineering/Books/` 로 미러링된다
(로컬 경로: `<vault>/public/resource/AI & Data & Engineering/Books/`).
책/챕터를 추가·수정하면 **미러에도 동일하게 반영하고 그쪽 repo에서 별도 commit & push** 한다.

미러 반영 규칙:
- **챕터 `.md` 파일**은 그대로 복사한다 (내용 동일).
- **README.md는 그대로 복사하지 말 것.** 미러 README는 **상대 경로**를 쓴다.
  즉 링크에서 `03_Resources/AI%20&%20Data%20&%20Engineering/Books/` 프리픽스를 제거한
  형태(예: `machine_learning_platform_engineering/01_...md`)로 넣는다.
  새 섹션만 이 상대 경로로 변환해 미러 README 최하단에 append 한다.
- **`CLAUDE.md`, `.claude/`, `.gitignore` 등 작업용 메타 파일은 미러에 복사하지 않는다.**

미러 동기화 검증 (양쪽 트리 비교 — README만 경로 프리픽스 차이로 다르게 나오는 게 정상):

```bash
SRC="<vault>/03_Resources/AI & Data & Engineering/Books"
DST="<vault>/public/resource/AI & Data & Engineering/Books"
# 콘텐츠 parity (README는 정규화 후 비교) — 비어 있으면 일치
diff <(sed 's#03_Resources/AI%20&%20Data%20&%20Engineering/Books/##g' "$SRC/README.md") "$DST/README.md"
# 파일 트리 parity — README 외 차이가 없어야 한다
diff -rq --exclude=.git --exclude=.claude --exclude=CLAUDE.md --exclude=.gitignore --exclude=.DS_Store "$SRC" "$DST"
```
