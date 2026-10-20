# Issue #9 에임 오프셋 구현

## Issue
- **Issue 번호:** #9
- **기능명:** 3인칭 에임 오프셋(Aim Offset) 및 상태별 애니메이션 분기
- **담당자: seok0317
- **담당 영역:** Animation / Math / PlayerAnimInstance / AnimGraph
- **관련 의존성:** #9 에임 오프셋 C++ (180도 스냅 방지 알고리즘), #8 조준 시스템 (상태 변수 연동), #6 3인칭 캐릭터 이동 구현 (자유 시점 모드)

## 목적
마우스(카메라)가 향하는 상하좌우 방향에 맞춰 캐릭터의 상체(척추, 머리 등)가 자연스럽게 따라가도록 에임 오프셋(Aim Offset) 데이터를 계산하고 렌더링한다.
특히, 자유 시점 상태에서 플레이어가 캐릭터의 등 뒤(얼굴 정면)를 바라보기 위해 마우스를 넘길 때 발생하는 180도 척추 스냅(Snapping)을 완벽하게 방지하는 '부드러운 반전 보간 알고리즘'을 C++로 구현한다. 또한, 캐릭터의 현재 상태(서기, 앉기, 조준, 달리기)에 따라 각기 다른 에임 오프셋 에셋을 동적으로 교체하고 하체의 이동(Locomotion)과 자연스럽게 합성(Layered Blend)하는 ABP 구조를 설계한다.

## 주요 구조
```text
UPlayerAnimInstance
 └─ UpdateAimOffsetValues() : 틱마다 AimPitch와 AimYaw를 부드럽게 계산
     ├─ AimPitch : 컨트롤러 상하 각도를 -90 ~ +90으로 보정
     └─ AimYaw : 컨트롤러와 액터의 좌우 각도 차이 보정 (180도 스냅 방지 반전 알고리즘 적용)
 └─ ABP_Player (AnimGraph)
     ├─ Blend Poses by bool (상태별 AO 에셋 교체)
     └─ Layered blend per bone (Cache_Locomotion + 상체 조준 포즈 합성)
```

## 구현 파일
- `Source/TeamProject_Fin/GameCore/Animation/PlayerAnimInstance.h & .cpp`
- `Content/.../AO_RifleAim`, `AO_RifleCrouch`, `AO_RifleHip`
- `Content/.../ABP_Player`

---

## 주요 구현 내용 및 시스템 역할

### 1. 상하 조준 (AimPitch) 계산 (C++)
- 컨트롤러가 바라보는 카메라의 상하 조준 각도(Pitch)를 추출한다.
- 회전값을 `FRotator::NormalizeAxis`를 사용하여 -180 ~ +180 값으로 정규화(Normalize)한 뒤, 위/아래 최대 고개 각도인 -90 ~ 90도로 클램프(Clamp)하여 안전한 `TargetPitch`를 산출한다.

### 2. 좌우 조준 (AimYaw) 180도 척추 스냅 방지 알고리즘 (C++)
- **문제점**: 자유 시점 모드에서 마우스를 우측 등 뒤(+179도)에서 좌측 등 뒤(-179도)로 살짝 넘길 때, 단순 Clamp(-90~90)에 의해 목표 방향이 단 1프레임 만에 +90도에서 -90도로 180도 순간이동하며 척추가 꺾이는 심각한 팝핑(Popping) 버그가 발생한다.
- **수학적 반전 보정 알고리즘 (Reflection Math)**:
  - 카메라가 캐릭터의 물리적 어깨 한계 각도(90도)를 넘어가는 순간, 캐릭터가 억지로 카메라를 쫓아가는 것을 포기하고 서서히 정면(0도)으로 원상 복구하도록 궤도를 반전시켰다.
  - 우측 초과 (`TargetYaw > 90`): `180.0f - TargetYaw` 적용.
  - 좌측 초과 (`TargetYaw < -90`): `-180.0f - TargetYaw` 적용.
  - **결과**: 등 뒤 정중앙 선을 넘나들어도 양쪽 모두 목표값이 0도(정면)로 부드럽게 수렴하여 척추 꺾임 현상을 원천적으로 차단했다. 
- 이후 `FMath::FInterpTo`를 이용해 목표 Pitch와 Yaw를 매 프레임 부드럽게 섞어(Lerp) 묵직한 에임 오프셋 데이터를 완성한다.

### 3. 상황별 에임 오프셋(AO) 에셋 분리 및 구성 (ABP)
캐릭터의 자세와 무기 파지법에 맞춰 3개의 전용 에임 오프셋 에셋을 9방향 2D 블렌드 스페이스(X: Yaw, Y: Pitch)로 구성했다.
- **`AO_RifleAim`**: 서 있는 상태에서 정조준(우클릭) 시 사용하는 기본 에임 오프셋.
- **`AO_RifleCrouch`**: 앉은 상태(`Is Crouching`)에서의 조준 및 지향 사격을 담당.
- **`AO_RifleHip`**: 비조준 상태에서 총구를 가볍게 내리거나 지향 사격(Hipfire) 자세를 취할 때 사용.

### 4. AnimGraph 상태별 우선순위 분기 및 상하체 분리 합성 (ABP)
C++에서 갱신되는 불리언 변수(`Is Crouching`, `Is Aiming`, `Is Jogging`)를 활용하여 상황에 맞는 AO를 우선순위에 따라 블렌딩한다.
- **상태 분기 (`Blend Poses by bool`)**:
  - 베이스: `AO_RifleHip`
  - `Is Crouching`이 True일 경우 `AO_RifleCrouch`로 자세를 덮어씌운다.
  - 그 위에서 `Is Aiming`이 True일 경우 최종적으로 `AO_RifleAim`이 적용되어 정조준 상태가 가장 높은 시각적 우선순위를 가진다.
- **상하체 분리 (`Layered blend per bone`)**:
  - 위에서 결정된 상체 조준 포즈를 하체의 이동 베이스(`Cache_Locomotion`)와 합성한다. 척추(Spine) 본을 기점으로 웨이트를 1.0 주어 독립적인 상/하체 애니메이션 재생을 완성했다.
- **전력 질주(Jogging) 예외 처리**:
  - 최상단 블렌드에 `Is Jogging` 노드를 배치하여 캐릭터가 달리기 시작하면 상체의 조준(AO) 상태를 완전히 무시하고 상하체 전체가 달리기 로코모션으로 강제 전환되도록 척추 뒤틀림을 방지했다.

---

## 테스트 방법

**애니메이션 트랜지션 및 보간 테스트**
- 서서 조준하다가 앉기 키를 누를 경우, 시선(Pitch, Yaw)은 그대로 유지된 채 상체 포즈만 `AO_RifleAim`에서 `AO_RifleCrouch`로 자연스럽게 전환되는지 확인 완료.
- 이동(WASD) 중 마우스를 빠르게 회전시켰을 때, 다리의 스텝이 꼬이지 않고 상체만 C++의 `FInterpTo` 속도에 맞춰 묵직하게 따라오는지 디버깅 완료.
- 전력 질주(Shift) 입력 순간 에임 오프셋 블렌딩이 부드럽게(Blend Time 0.1) 해제되며 달리기 모션으로 이행되는지 확인 완료.

**자유 시점 한계 돌파 테스트 (팝핑 검증)**
- 가만히 서 있거나 앉아 있는 상태(자유 시점)에서 마우스를 돌려 캐릭터의 등 뒤(얼굴 정면)를 쳐다보며 마우스를 비볐을 때, 허리가 180도로 튀지 않고 앞을 향해 자연스럽게 고개를 두고 있는지 수학적 반전 알고리즘 검증 완료.

## 완료 조건 확인
- [x] 컨트롤러와 액터 회전의 델타(Delta) 추출 및 `AimPitch` 정규화/Clamp 로직 작성
- [x] 자유 시점 180도 팝핑을 막는 반전(Reflection) 수학 알고리즘 적용 (`AimYaw`)
- [x] `FInterpTo`를 이용한 부드러운 상하좌우 데이터 보간 적용
- [x] 9방향 2D 블렌드 스페이스 기반의 상황별 AO 에셋 3종(Aim, Crouch, Hip) 세팅
- [x] 상태 변수(`Is Crouching`, `Is Aiming`)를 활용한 AO 에셋 동적 우선순위 교체 구축
- [x] `Layered blend per bone`을 이용한 하체(로코모션)와 상체(AO) 독립 재생 합성
- [x] 달리기(`Is Jogging`) 진입 시 조준 모션 강제 무효화 처리
