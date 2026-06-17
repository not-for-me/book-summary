# 14. Collaboration

## 챕터 개요 (3줄 요약)
- 파이썬은 명확한 인터페이스 경계를 가진 API를 구성하는 언어 기능과 협업을 위한 모범 사례·표준 도구를 제공한다.
- 가상 환경(venv), 도큐스트링, 패키지, 루트 예외 등으로 유지보수 가능하고 안정적인 코드를 작성한다.
- 순환 의존성 해결, warnings 마이그레이션, typing 정적 분석, 오픈소스 번들링으로 대규모 협업을 원활히 한다.

---

## Item 116: 커뮤니티 빌드 모듈을 찾는 곳을 알라 (Know Where to Find Community-Built Modules)
> PyPI는 커뮤니티가 만든 풍부한 패키지의 중앙 저장소다.

- Python Package Index(PyPI, https://pypi.org)에 수많은 패키지가 있다.
- `python3 -m pip install`로 패키지를 설치한다.
- 대부분의 PyPI 모듈은 무료·오픈소스다.
- 각 모듈은 고유한 라이선스를 가진다.
- venv와 함께 쓰면 프로젝트별 패키지를 추적할 수 있다.

```bash
python3 -m pip install numpy
```

## Item 117: 격리되고 재현 가능한 의존성엔 가상 환경을 사용하라 (Use Virtual Environments)
> venv는 충돌 없이 같은 패키지의 여러 버전을 설치·재현하게 한다.

- 전역 설치는 전이적 의존성 충돌(dependency hell)을 일으킬 수 있다.
- `python -m venv`로 격리된 환경을 만들고 `source bin/activate`로 활성화한다.
- `deactivate`로 비활성화한다.
- `pip freeze > requirements.txt`로 의존성을 저장한다.
- `pip install -r requirements.txt`로 환경을 재현한다.

```bash
python3 -m venv myproject
source myproject/bin/activate
pip freeze > requirements.txt          # 의존성 저장
pip install -r requirements.txt        # 재현
```

## Item 118: 모든 함수·클래스·모듈에 도큐스트링을 작성하라 (Write Docstrings for Every Function, Class, and Module)
> 동적 언어인 파이썬에서 도큐스트링은 런타임에 접근 가능한 중요한 문서다.

- `def` 직후 삼중 따옴표 문자열로 도큐스트링을 작성한다(`__doc__`로 접근).
- 모듈은 첫 줄에 목적, 이후 주요 클래스·함수를 소개한다.
- 클래스는 동작·주요 속성·서브클래스 지침을 문서화한다.
- 함수는 인자·반환값·발생 예외를 설명한다.
- 타입 어노테이션이 있으면 도큐스트링에서 타입 정보는 생략해 중복을 피한다.

```python
def find_anagrams(word: str, dictionary: Container[str]) -> list[str]:
    """Find all anagrams for a word.
    Args:
        word: Target word.
        dictionary: All known actual words.
    Returns:
        Anagrams that were found.
    """
```

## Item 119: 패키지로 모듈을 조직하고 안정적 API를 제공하라 (Use Packages to Organize Modules and Provide Stable APIs)
> 패키지는 네임스페이스 분리와 외부 소비자를 위한 안정적 API를 제공한다.

- `__init__.py`를 디렉터리에 두면 패키지가 되어 다른 파일을 import할 수 있다.
- 패키지는 같은 파일명을 충돌 없이 고유 네임스페이스로 분리한다.
- `as` 절로 이름 충돌을 피한다.
- `__all__`로 공개 API 이름을 명시해 내부 구현을 숨긴다.
- `from x import *`는 출처를 숨기고 이름을 덮어쓰므로 피한다.

```python
# __init__.py
__all__ = []
from .models import *
__all__ += models.__all__
```

## Item 120: 배포 환경 구성엔 모듈 스코프 코드를 고려하라 (Consider Module-Scoped Code to Configure Deployment Environments)
> 모듈 스코프의 일반 파이썬 코드로 배포 환경별 동작을 다르게 정의한다.

- 프로덕션·개발 환경은 가정과 구성이 다르다.
- 모듈 스코프(함수 밖)의 if 문으로 환경별 이름을 정의한다.
- `__main__`의 TESTING 상수로 동작을 분기한다.
- `sys.platform`, `os.environ`으로 호스트를 검사해 정의를 맞춘다.
- 복잡해지면 configparser 같은 설정 파일로 분리한다.

```python
import sys
if sys.platform.startswith("win32"):
    Database = Win32Database
else:
    Database = PosixDatabase
```

## Item 121: 루트 예외를 정의해 호출자를 API로부터 격리하라 (Define a Root Exception to Insulate Callers from APIs)
> 모듈의 루트 예외 클래스로 호출자가 의도된 예외를 쉽게 잡고 격리하게 한다.

- 내장 예외(ValueError) 대신 모듈 고유 예외 계층을 정의하는 것이 강력하다.
- 루트 `Error` 클래스를 두고 모든 예외가 이를 상속한다.
- 호출자는 루트 예외를 잡아 API 오용을 인지한다.
- `Exception` 베이스를 잡으면 API 구현의 버그를 찾는다.
- 중간 루트 예외로 향후 더 구체적인 예외를 깨짐 없이 추가한다.

```python
class Error(Exception):
    """Base-class for all exceptions raised by this module."""
class InvalidDensityError(Error):
    """There was a problem with a provided density value."""
```

## Item 122: 순환 의존성을 끊는 법을 알라 (Know How to Break Circular Dependencies)
> 동적 import가 순환 의존성을 끊는 가장 간단한 해법이다.

- 두 모듈이 import 시점에 서로를 호출하면 순환 의존성으로 시작 시 충돌한다.
- 원인: 모듈 속성은 코드 실행(5단계) 후에야 정의되는데 import는 그 전에 가능하다.
- 최선책: 상호 의존성을 의존성 트리 하단의 별도 모듈로 리팩터링한다.
- import 재정렬은 PEP 8에 반하고 취약하다.
- import-configure-run 패턴이나 함수 내 동적 import로 해결한다.

```python
def show():
    import app   # 동적 import로 순환 끊기
    save_dialog.save_dir = app.prefs.get("save_dir")
```

## Item 123: 사용 리팩터링·마이그레이션엔 warnings를 고려하라 (Consider warnings to Refactor and Migrate Usage)
> warnings로 협업자에게 사용 중단 예정 API를 알려 코드 마이그레이션을 유도한다.

- warnings는 의존 라이브러리 변경을 사람에게 알리는 프로그래밍 방식이다.
- `warnings.warn(..., DeprecationWarning)`으로 사용 중단을 경고한다.
- `stacklevel`로 올바른 호출 위치를 보고한다.
- `-W error`로 경고를 에러로 만들어 테스트에서 회귀를 잡는다.
- 프로덕션에선 `logging.captureWarnings`로 경고를 로그에 복제한다.

```python
def require(name, value, default):
    if value is not None:
        return value
    warnings.warn(f"{name} will be required soon, update your code",
                  DeprecationWarning, stacklevel=3)
    return default
```

## Item 124: typing을 통한 정적 분석으로 버그를 없애라 (Consider Static Analysis via typing to Obviate Bugs)
> 타입 힌트와 정적 분석 도구로 런타임 전에 흔한 버그를 잡는다.

- `typing` 모듈과 어노테이션 문법으로 변수·함수에 타입 정보를 단다.
- 점진적 타이핑(gradual typing)으로 코드베이스를 점진적으로 업데이트한다.
- mypy(--strict), pyright 등 도구가 타입 정보로 버그를 탐지한다.
- 제네릭(`TypeVar`), 옵션 타입(`int | None`)으로 다양한 버그를 잡는다.
- 전방 참조는 문자열 어노테이션("SecondClass")으로 해결한다.

```python
def subtract(a: int, b: int) -> int:
    return a - b
subtract(10, "5")   # mypy가 타입 불일치를 사전 탐지
```

## Item 125: 파이썬 프로그램 번들링은 zipimport/zipapp보다 오픈소스 프로젝트를 선호하라 (Prefer Open Source Projects for Bundling)
> 데이터 파일·확장 모듈 문제 때문에 zip 번들링보다 Pex 같은 도구가 낫다.

- 파이썬은 zip 아카이브에서 모듈을 직접 로드할 수 있다(`zipimport`, `zipapp`).
- 압축 로딩의 성능 차이는 사실상 0이다.
- 그러나 데이터 파일 접근이 zip에서 깨진다(`pkgutil`로 일부 우회).
- 네이티브 확장 모듈은 OS 제약으로 zip에서 import할 수 없다.
- Pex, Shiv, PyInstaller 같은 오픈소스 도구가 이 문제들을 해결한다.

```bash
pex django_project -o myapp.pex   # 데이터 파일·네이티브 모듈 문제 해결
./myapp.pex -m manage check
```

---

## Summary (핵심 정리)
- PyPI와 venv로 의존성을 관리하고, 도큐스트링·패키지·루트 예외로 명확하고 안정적인 API를 제공한다.
- 모듈 스코프 코드로 배포 환경을 구성하고, 순환 의존성은 동적 import 등으로 끊는다.
- warnings로 마이그레이션을 유도하고, typing 정적 분석으로 버그를 예방하며, 번들링엔 Pex 같은 오픈소스 도구를 쓴다.
