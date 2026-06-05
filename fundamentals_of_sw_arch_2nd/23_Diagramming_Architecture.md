# 23. Diagramming Architecture

## 챕터 개요 (3줄 요약)

- 다이어그램은 아키텍트의 핵심 소통 기술로, 아무리 뛰어난 아이디어도 전달 못 하면 실현되지 않는다.
- 표현 일관성(representational consistency)과 적절한 도구·표준(UML, C4, ArchiMate)을 활용한다.
- 제목·선·색·키 등 다이어그램 지침을 따르되 초기엔 저비용 임시 산출물을 쓴다.

---

## 1. Representational Consistency (표현 일관성)

> 뷰를 바꾸기 전에 항상 부분이 전체 아키텍처 안에서 차지하는 위치(관계)를 먼저 보여준다.

전체 토폴로지 → 특정 부분과의 관계 → 세부 구조 순으로 드릴다운한다. 위치를 안 보여주면 혼란을 준다.

---

## 2. Diagramming Tools (도구)

> 강력한 도구를 깊이 익히되, 설계 초기엔 저비용 임시 산출물을 쓴다.

### Irrational Artifact Attachment (비합리적 산출물 애착)

산출물 제작 시간에 비례해 비합리적 애착이 커진다(Visio로 4시간 들인 다이어그램에 더 집착). Agile은 적은 의식의 just-in-time 산출물 선호(인덱스 카드·스티키노트로 쉽게 버리고 실험). 화이트보드 사진 대신 태블릿+프로젝터(무한 캔버스·복붙·디지털화·원격 협업). 충분히 반복한 뒤 정교한 도구로 옮긴다.

권장 기능: Layers(레이어 — 의미적으로 사용: 기반 레이어=토폴로지, 다음 레이어=구현 세부), Stencils/Templates(공통 시각 컴포넌트 라이브러리로 일관성), Magnets(선 자동 스냅).

---

## 3. Diagramming Standards (표준: UML, C4, ArchiMate)

> 업계의 세 인기 표준이 있다.

- UML(Unified Modeling Language): Booch·Jacobson·Rumbaugh가 1980년대 통합. 위원회 설계로 큰 영향은 못 줌. 클래스·시퀀스 다이어그램만 주로 생존.
- C4: Simon Brown(2006~2011)이 UML 결함 보완. 네 C: Context(전체 맥락·사용자·외부 의존), Container(배포 경계·컨테이너 — 운영팀과 접점), Component(컴포넌트 뷰 — 아키텍트 관점), Class(UML 클래스 다이어그램 재사용). 생태계가 활발하고 도구 지원 많음.
- ArchiMate: The Open Group의 오픈소스 전사 아키텍처 모델링 언어. "가능한 한 작게"가 목표인 경량 언어.

---

## 4. Diagram Guidelines (다이어그램 지침)

> 자신만의 다이어그램 스타일을 만들되 일반 지침을 따른다.

- Titles: 모든 요소에 제목(잘 알려진 것 제외).
- Lines: 충분히 두껍게, 방향은 화살표로. 표준: 실선=동기 통신, 점선=비동기 통신.
- Shapes: 표준 도형 부재 — 자체 세트 구성(3D 박스=배포 산출물, 사각형=컨테이너, 원통=DB).
- Labels: 모호함 가능성 있으면 모두 라벨링.
- Color: 충분히 활용하되 색맹 고려해 색+고유 아이콘 병행.
- Keys: 도형이 모호하면 키(범례) 포함 — 오해 소지 있는 다이어그램은 없느니만 못함.

```
  Standard convention:
   solid line  ----->  synchronous communication
   dotted line ---->  asynchronous communication
```

---

## Summary (핵심 정리)

- 다이어그램 표준은 일관된 소통을 제공하지만, 표준이 부족할 땐 합리적 예외를 허용한다.
- 표현 일관성으로 맥락을 보여주고, 레이어를 의미적으로 사용하며 적절한 표준(C4 등)을 택한다.
- 초기엔 경량·임시 산출물을 쓰고, 산출물에 과도하게 애착하지 않도록 객관성을 유지한다.
