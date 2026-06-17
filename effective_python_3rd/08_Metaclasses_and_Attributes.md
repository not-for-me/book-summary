# 08. Metaclasses and Attributes

## 챕터 개요 (3줄 요약)
- 메타클래스(metaclass)는 class 문을 가로채 클래스 정의 시점에 특별한 동작을 부여한다.
- 동적 속성 접근(@property, 디스크립터, __getattr__ 등)으로 단순 클래스를 점진적으로 정교하게 만들 수 있다.
- 강력한 기능엔 함정이 따르므로 최소 놀람 원칙을 따르고, 메타클래스보다 __init_subclass__·클래스 데코레이터 같은 단순한 도구를 선호한다.

---

## Item 58: setter/getter 메서드 대신 평범한 속성을 사용하라 (Use Plain Attributes Instead of Setter and Getter Methods)
> 파이썬에서는 명시적 getter/setter 대신 공개 속성으로 시작하고 필요 시 @property로 전환한다.

- 명시적 getter/setter는 비파이썬적이고 in-place 연산이 번거롭다.
- 단순 공개 속성으로 시작하는 것이 자연스럽다.
- 특별한 동작이 필요하면 `@property`와 `@x.setter`로 전환한다.
- setter에서 타입 검사·검증·불변성을 구현할 수 있다.
- @property 메서드는 빠르고 부작용이 없어야 한다(느린 작업은 일반 메서드로).

```python
class BoundedResistance(Resistor):
    @property
    def ohms(self):
        return self._ohms
    @ohms.setter
    def ohms(self, ohms):
        if ohms <= 0:
            raise ValueError(f"ohms must be > 0; got {ohms}")
        self._ohms = ohms
```

## Item 59: 속성 리팩터링 대신 @property를 고려하라 (Consider @property Instead of Refactoring Attributes)
> @property로 기존 속성에 새 동작을 부여해 호출부 수정 없이 데이터 모델을 점진 개선한다.

- @property로 단순 속성을 즉석 계산으로 전환할 수 있다.
- 모든 기존 사용처를 다시 쓰지 않고 새 동작을 마이그레이션한다.
- 인터페이스를 시간에 걸쳐 개선하는 임시방편 역할을 한다.
- 더 나은 데이터 모델로의 점진적 진전을 가능케 한다.
- @property를 과도하게 확장하게 되면 클래스를 리팩터링할 때다.

```python
class NewBucket:
    @property
    def quota(self):                       # 즉석 계산
        return self.max_quota - self.quota_consumed
    @quota.setter
    def quota(self, amount):
        ...   # 기존 fill/deduct 인터페이스와 호환
```

## Item 60: 재사용 가능한 @property 메서드엔 디스크립터를 사용하라 (Use Descriptors for Reusable @property Methods)
> 디스크립터(descriptor)는 여러 속성과 클래스에서 검증 로직을 재사용하게 한다.

- @property 메서드는 같은 클래스의 여러 속성이나 다른 클래스에서 재사용할 수 없다.
- 디스크립터는 `__get__`/`__set__`을 구현해 속성 접근을 가로챈다.
- 단일 디스크립터 인스턴스를 공유하면 인스턴스 간 상태가 섞이는 버그가 생긴다.
- `__set_name__`으로 속성별 내부 이름을 계산해 인스턴스 딕셔너리에 저장한다.
- 이로써 메모리 누수 없이 검증 로직을 재사용한다.

```python
class NamedGrade:
    def __set_name__(self, owner, name):
        self.internal_name = "_" + name
    def __set__(self, instance, value):
        if not (0 <= value <= 100):
            raise ValueError("Grade must be between 0 and 100")
        setattr(instance, self.internal_name, value)
```

## Item 61: 지연 속성엔 __getattr__, __getattribute__, __setattr__을 사용하라 (Use __getattr__, __getattribute__, and __setattr__ for Lazy Attributes)
> 이 훅들로 속성을 지연 로딩·저장하는 제네릭 코드를 작성할 수 있다.

- `__getattr__`은 인스턴스 딕셔너리에 없는 속성 접근 시에만 호출된다.
- `__getattribute__`는 모든 속성 접근마다 호출되어 트랜잭션 검사 등에 쓰인다.
- 없어야 할 속성엔 `AttributeError`를 발생시킨다.
- `__setattr__`은 모든 속성 할당마다 호출된다.
- 무한 재귀를 피하려면 `super().__getattribute__`/`super().__setattr__`를 호출한다.

```python
class LazyRecord:
    def __getattr__(self, name):
        value = f"Value for {name}"
        setattr(self, name, value)   # 다음 접근부턴 캐시 사용
        return value
```

## Item 62: __init_subclass__로 서브클래스를 검증하라 (Validate Subclasses with __init_subclass__)
> 메타클래스보다 __init_subclass__로 서브클래스 정의 시점에 검증하는 것이 명료하다.

- 메타클래스는 `type`을 상속하고 `__new__`에서 클래스 정보를 검사·수정한다.
- 검증을 import 시점처럼 이른 시기에 수행해 에러를 빨리 잡는다.
- `__init_subclass__`(3.6+)는 메타클래스 없이 동일 검증을 더 간결하게 한다.
- 메타클래스는 클래스당 하나만 지정 가능해 합성이 어렵다.
- `super().__init_subclass__()`를 호출하면 다중 계층·다중 상속에서 합성 가능한 검증이 된다.

```python
class BetterPolygon:
    sides = None
    def __init_subclass__(cls):
        super().__init_subclass__()
        if cls.sides < 3:
            raise ValueError("Polygons need 3+ sides")
```

## Item 63: __init_subclass__로 클래스 존재를 등록하라 (Register Class Existence with __init_subclass__)
> 자동 클래스 등록으로 역방향 조회(식별자→클래스)를 안전하게 구현한다.

- 등록은 직렬화·ORM·플러그인·콜백 훅 등 역방향 조회에 유용하다.
- 수동 `register_class` 호출은 빠뜨리기 쉬워 런타임 오류를 낸다.
- 메타클래스의 `__new__`에서 자식 클래스를 자동 등록할 수 있다.
- `__init_subclass__`로 같은 동작을 더 간결하게 구현한다.
- 상속 트리만 맞으면 등록 누락이 절대 없도록 보장된다.

```python
class BetterRegisteredSerializable(BetterSerializable):
    def __init_subclass__(cls):
        super().__init_subclass__()
        register_class(cls)   # 자동 등록
```

## Item 64: __set_name__으로 클래스 속성을 주석하라 (Annotate Class Attributes with __set_name__)
> __set_name__으로 디스크립터가 자신이 할당된 속성 이름을 알게 해 중복을 없앤다.

- 디스크립터에 속성 이름을 중복 전달(`Field("first_name")`)하는 것은 불필요하다.
- 메타클래스로 클래스 정의 시 디스크립터에 이름을 주입할 수 있다.
- 그러나 메타클래스 방식은 특정 부모 상속을 강제한다.
- `__set_name__`(3.6+)은 클래스 정의 시 owner와 속성 이름을 받는다.
- 메타클래스 없이 디스크립터가 주변 클래스와 속성 이름을 인식한다.

```python
class Field:
    def __set_name__(self, owner, column_name):
        self.column_name = column_name
        self.internal_name = "_" + column_name
```

## Item 65: 속성 간 관계 설정에 클래스 본문 정의 순서를 고려하라 (Consider Class Body Definition Order)
> 클래스 본문의 정의 순서가 __dict__에 보존됨을 이용해 CSV 열 매핑 등을 구현한다.

- 클래스 본문의 속성·메서드는 런타임에 `__dict__`로 검사할 수 있다.
- `__dict__`는 정의 순서를 보존한다(딕셔너리 삽입 순서 보존).
- `__init_subclass__` + `__dict__` 검사로 필드명을 자동 수집한다.
- 디스크립터와 결합해 검증·변환 기능을 추가할 수 있다.
- 메서드 데코레이터로 워크플로 단계 순서를 발견하는 데도 쓸 수 있다.

```python
class BetterRowMapper(RowMapper):
    def __init_subclass__(cls):
        fields = [k for k, v in cls.__dict__.items() if v is Ellipsis]
        cls.fields = tuple(fields)   # 정의 순서 = CSV 열 순서
```

## Item 66: 합성 가능한 클래스 확장엔 메타클래스보다 클래스 데코레이터를 선호하라 (Prefer Class Decorators over Metaclasses)
> 클래스 데코레이터는 충돌 없이 합성되어 모든 메서드를 수정하는 데 적합하다.

- 모든 메서드를 데코레이트하려면 일일이 재정의하는 보일러플레이트가 생긴다.
- 메타클래스로 자동화할 수 있으나 클래스당 하나만 허용되어 합성이 안 된다.
- 클래스 데코레이터는 클래스를 받아 수정·재생성 후 반환하는 함수다.
- 기존 메타클래스가 있어도 함께 동작한다.
- 여러 유틸리티를 충돌 없이 합성할 수 있어 클래스 확장에 최적이다.

```python
def trace(klass):
    for key in dir(klass):
        value = getattr(klass, key)
        if isinstance(value, TRACE_TYPES):
            setattr(klass, key, trace_func(value))
    return klass

@trace
class DecoratedTraceDict(dict):
    pass
```

---

## Summary (핵심 정리)
- 속성은 공개로 시작해 필요 시 @property·디스크립터로 전환하고, 지연 속성엔 __getattr__/__setattr__ 훅을 쓴다.
- 서브클래스 검증·등록·속성 주석은 무거운 메타클래스보다 __init_subclass__와 __set_name__으로 간결하게 구현한다.
- 클래스 본문 정의 순서를 활용하고, 합성 가능한 확장엔 메타클래스보다 클래스 데코레이터를 선호한다.
