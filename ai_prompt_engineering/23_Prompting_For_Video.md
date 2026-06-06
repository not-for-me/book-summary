# 23. Prompting for Video

## 챕터 개요 (3줄 요약)

- 비디오 생성은 이미지 프롬프팅을 시간 차원으로 확장한다 — 비디오를 만드는 모델은 주체가 프레임마다 그럴듯하게 움직이는 시퀀스를 내야 하고, 시각 내러티브가 믿을 만한 영상으로 읽히려면 연속성이 클립 전반에서 유지돼야 한다.
- 모션이 모델이 작동할 수 있는 언어로 어떻게 기술되는지와, 일관성(coherence)이 어떻게 명시적으로 요청할 무언가가 되는지를 다루며 생성 미디어 검토를 마무리한다.
- 그 새 차원을 위한 작업 어휘를 제공해 프롬프트가 처음부터 끝까지 일관되게 유지되는 시퀀스를 내게 한다.

---

## 1. Describing Motion and Action

> 모션은 비디오를 정지 이미지와 구별하는 변수이며, 모션을 기술하는 단어가 모델이 내는 것을 형성한다 — 유용한 프롬프트는 무엇이 움직이고 어떤 종류의 모션이 수행되는지를 명명한다.

- 적용지침: 프롬프트는 주변 세계가 그 모션에 어떻게 반응하는지도 가리킨다. 프롬프트가 이 요소를 과소 명시하면 모델이 기본 행동으로 간극을 채우고, 그 기본 행동은 흔히 뻣뻣하거나 프레임마다 같은 마이크로 동작을 반복한다.
- 적용지침: Geng et al. (2025)는 비디오 디퓨전 모델이 모션 프롬프트로 작동하는 모션 궤적(motion trajectory)에 조건화될 수 있음을 입증하고, 상위 요청이 더 밀집한 모션 설명으로 확장될 때 믿을 만한 화면 행동으로 더 신뢰성 있게 번역됨을 보였다 — "a dog runs"가 동물 모션의 보폭·속도를 명시한 문구보다 덜 일관된 영상을 낸다. 모션 설명이 밀집할수록 모델이 추측할 가능한 애니메이션 공간이 작아진다.
- 적용지침: 유용한 분할을 기억하라 — subject motion(프레임 내 인물·객체가 하는 것), camera motion(클립 전반 시점의 행동), environmental dynamics(부차 효과 — 움직이는 잎, 발걸음에 날리는 먼지). 성숙한 프롬프트는 각 계층을 별도로 다뤄 모델이 해결할 모호성을 줄인다.
- 적용지침: 비디오 디퓨전의 모션 통제는 구조화 조건화 신호와, 손으로 설계한 프롬프트가 모션을 별개 계층으로 분리하는 것에서 크게 이득을 본다. 모션 지속 시간(motion duration)에 명시적인 것이 큰 도움 — 행동을 측정 가능한 시간 범위에 묶는 문구가 모델에 계획할 시간 타깃을 줘, 모델이 중간에 얼어붙거나 전체 모션을 클립 첫 1초에 몰아넣을 가능성을 줄인다.
- 적용지침: action verb는 추상 명사가 못 하는 정보를 담는다 — "the boy jumps"가 "the boy crouches and pushes off the ground with both feet"보다 모델에 작업할 것을 덜 준다. 물리적 시퀀싱에 근거한 구체 동사가 디퓨전 과정이 보간할 중간 자세를 공급해, 결과 모션이 은유적·추상적 언어가 내는 것보다 훨씬 믿을 만해진다. 비디오 프롬프트를 처음 쓸 때 같은 행동을 옆에서 보는 아이에게 묘사할 동사를 먼저 떠올려라.
- 적용지침 예시: "A golden retriever sprints across a meadow at moderate speed, with its head held low and tail extended behind the body. Afternoon breeze bends the surrounding stalks in the same direction the dog is traveling."

---

## 2. Temporal Coherence and Consistency

> 비디오의 일관성(coherence)은 시청자가 한 프레임에서 다음 프레임으로 같은 주체를 혼동 없이 인식하게 하는 속성이다 — 클립의 정체성이 시간에 걸쳐 유지되고 조명·팔레트가 지속 안정해야 한다.

- 적용지침: 일관성에 실패하는 모델은 drift(주체 외양이 클립 동안 점진 변화), flicker(안정해야 할 표면의 고주파 변동), identity transition(이유 없이 캐릭터가 다른 사람으로 변형)을 낸다.
- 적용지침: 분할 cross-attention 메커니즘이 긴 클립의 각 세그먼트에 프레임 전반 지속되는 내부 표현을 줘 이 실패를 줄인다 — 프롬프팅 함의: 주체를 뚜렷한 특징을 잠그는 방식으로 기술하면 모델이 생성의 롤링 윈도우에 걸쳐 붙잡을 정보를 더 많이 가진다. 생생하고 구체적인 설명이 비디오 프롬프트에서 일관성을 돕는다.
- 적용지침: 조명 안정성(lighting stability)은 자체 문제 — 조명을 한 번 명명하고 나머지 장면 설명이 조명을 암묵적으로 처리하게 두면 색온도가 마지막 프레임까지 표류하는 클립이 나온다. 하루 중 시간과 키 라이트 방향을 명시하면 이 표류가 줄어드는데, 모델이 그 조명 사실을 같은 프롬프트가 생성한 모든 프레임에 걸친 제약으로 다루기 때문이다.
- 적용지침: 팔레트 일관성도 유사 논리 — 채도 높은 파란 하늘로 시작해 바랜 회색으로 끝나는 클립은 프레임 내 행동과 무관하게 표류한 것이다. 프레임 두 영역의 지배 색을 명명하면(imprecise 색 언어로도) 모델이 컬러 그레이딩을 잠그는 데 도움. "전경의 따뜻한 초록 풀과 배경의 부드러운 회색 하늘" 같은 문구가 클립의 색 외피를 첫 프레임에서 마지막까지 제약하는 실제 일을 한다.
- 적용지침 예시: "Maintain the same woman throughout the clip. She has dark curly hair tied back and wears a navy linen jacket over a white shirt. She sits at a wooden cafe table reading a paperback. Soft morning light comes from camera right with warm tones across the scene. Hold this lighting and her appearance for the full duration."

---

## 3. Camera Language and Cinematic Direction

> 영화 촬영의 어휘가 시점이 무엇을 해야 하는지 기술할 정밀한 방법을 준다.

- 적용지침: He et al. (2024b)는 CameraCtrl을 도입했다 — 비디오 디퓨전 모델을 명시적 카메라 포즈 궤적에 조건화해 생성 클립이 공간 내 지정 경로를 따르게 하는 방법. 학습된 카메라 임베딩이 영화 촬영 용어를 정확한 시점 행동으로 번역함을 보였다 — 프롬프팅 함의: 모델이 특정 포즈 패턴과 연관 학습한 용어는 신뢰할 카메라 무브를, 부정확한 용어는 모호한 것을 낸다.
- 적용지침: 알아둘 카메라 언어 범주 — shot type(주체로부터 카메라 거리; close-up은 고립 세부, wide shot은 주변 환경, medium shot은 대화 장면의 표준 프레이밍), camera movement(공간 내 카메라 경로; dolly는 주체로/에서 물리적 이동, pan은 수평축 회전), pacing 단서(무브가 클립 동안 얼마나 천천히/빠르게 펼쳐질지).
- 적용지침: He et al. (2024b)는 명시적 포즈 조건화가 실험에서 가장 정확한 카메라 통제를 냄을 발견했다 — 자연어 프롬프팅에 대한 함의: "the camera moves around the scene" 같은 모호한 지시보다 구체 용어가 더 신뢰할 결과를 낸다. 무브와 지속 시간을 함께 명명하면 무브만 명명하는 것보다 통제가 커진다("slow dolly-in over five seconds"가 "a slow zoom"보다 훨씬 통제된 푸시를 낸다).
- 적용지침: 렌즈 선택·depth of field가 또 다른 영화적 통제 계층 — 24mm 광각 렌즈는 늘어난 확장 프레임을, 50mm 렌즈는 자연 인간 시각 관점을 근사한다. 상상된 렌즈의 초점 거리 명시가 생성 이미지 기하에 측정 가능한 효과를 갖는다(모델이 초점 거리 용어를 특징적 왜곡과 연관 학습). depth of field는 "shallow depth of field" 명명이나 "wide aperture" 참조로 요청 가능하며 둘 다 모델을 cinema 연관 soft-background 룩으로 민다.
- 적용지침 예시: "Slow dolly-in over six seconds toward a wooden door at the end of a dim corridor. The camera holds steady at chest height. Final framing brings the door's brass handle to the center of the frame."

---

## 4. Structuring Prompts for Sequences and Scenes

> 단일 클립 프롬프트는 모델에 완료할 작업 하나를 주지만, 시퀀스(sequence)는 여러 연결된 작업을 연달아 처리하며 샷 경계를 가로질러 연속성을 유지하게 요청한다.

- 적용지침: 도전은 각 새 샷이 신선한 생성이며 그 신선한 생성이 이전 샷과 상태를 native하게 공유 안 한다는 것이다. Kara et al. (2025)는 ShotAdapter를 도입했다 — 전환 토큰(transition token)과 로컬 어텐션 마스킹 전략으로 text-to-video 디퓨전을 멀티 샷 출력으로 확장해 샷 특정 프롬프팅을 허용하는 프레임워크. 결과는 샷이 이산적으로 전환되면서 같은 캐릭터·설정이 컷 전반에서 일관되게 유지되는 클립이다.
- 적용지침: 계층적 캡션(hierarchical caption)이 캐스트·환경에 대한 불변 사실을 담은 전역 프롬프트(global prompt)를 유지하고, 시퀀스 내 각 샷이 그 샷에 특정한 행동·카메라 무브를 기술하는 자체 per-shot 프롬프트를 갖는다. 위계가 무엇이 전체 시퀀스에 걸쳐 유지돼야 하고 무엇이 개별 샷 안에서 변해도 되는지를 모델에 알린다.
- 적용지침: 실무 권고 — 전역 프롬프트를 먼저 쓰고, 시퀀스 각 세그먼트의 per-shot 프롬프트를 쓰며, 세그먼트 사이에 명확한 샷 경계 마커를 둬라. 전역 프롬프트는 시퀀스 전반 안정해야 할 모든 것을 고정하고, per-shot 프롬프트는 그 샷에 변해야 할 요소만 바꿔야 한다 — 이 분리가 모델의 인지 부하를 줄이고 가시적으로 더 일관된 멀티 샷 출력을 낸다.
- 적용지침: 멀티 샷 일관성은 연구 문헌의 미해결 문제로 남아 있다 — 모델은 한 카메라 무브 동안 주체를 비교적 쉽게 유지하지만, 하드 컷으로 새 샷에 주체를 유지하는 것은 현저히 어렵고 프롬프트·기저 아키텍처 양쪽의 상당한 도움을 요한다.
- 적용지침: 전환 타입(transition type)도 프롬프트 어휘의 일부 — 하드 컷(hard cut)은 두 샷 간 즉각 변화로 대부분 내러티브 비디오의 기본값이고, 디졸브·페이드 같은 소프트 전환은 추가 프롬프팅을 요하며 현재 모델에서 덜 신뢰할 만하다(훈련 데이터가 하드 컷을 선호하므로).
- 적용지침: scene-level과 sequence-level 프롬프팅의 관계 — 장면(scene)은 단일 설정에서 일어나는 연속 행동 블록(여러 샷으로 구성돼도)이고, 시퀀스(sequence)는 장면 사이를 이동하는 더 긴 호다. 현재 모델은 전역 프롬프트가 위치·캐스트를 안정 유지하면 scene-level 일관성을 합리적으로 처리하지만, 같은 캐릭터가 두 다른 설정에 나타나는 sequence-level 일관성은 더 까다로워 위의 아키텍처 지원이 결정 요인이 된다.
- 적용지침 예시: "[Global] A bearded man in a brown wool coat stands near a snow-covered cabin at dusk. Cool blue ambient light from the sky; warm orange firelight from a window behind him. Style: cinematic, 35mm film grain, shallow depth of field. [Shot 1, three seconds] Wide shot from the front. Slow push-in toward the man. He pulls his collar tighter against the wind. [Shot cut] [Shot 2, four seconds] Medium close-up from his right side. He turns his head toward the cabin window, and the firelight catches the side of his face."

---

## Summary (핵심 정리)

- 모션을 subject·camera·environmental 계층으로 분리하고 구체 action verb와 측정 가능한 지속 시간으로 밀집하게 기술해, 모델이 추측할 애니메이션 공간을 좁혀라(Geng 2025).
- 시간 일관성을 위해 주체를 뚜렷한 특징으로 기술하고 조명·팔레트(두 영역의 지배 색)를 전체 지속 시간에 걸쳐 유지하도록 명시해 drift·flicker를 막아라.
- 카메라 무브와 지속 시간을 함께 명명하고(CameraCtrl, He 2024b), 멀티 샷은 전역 프롬프트(불변 사실)+per-shot 프롬프트+명확한 샷 경계 마커의 계층 구조로 작성하라(ShotAdapter, Kara 2025).
