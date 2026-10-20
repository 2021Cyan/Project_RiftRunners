# Issue #8 조준 시스템 구현

## Issue
- **Issue 번호:** #8
- **기능명:** 조준(Aiming) 시스템 구현 (C++ 최적화 및 GAS 데이터 세팅 연동)
- **담당자: seok0317
- **담당 영역:** Character / Input / Gameplay Ability System (GAS) / AnimGraph
- **관련 의존성:** #6 3인칭 캐릭터 이동 구현, #7 TPS 카메라 구현

## 목적
플레이어가 무기를 조준할 때 캐릭터의 상태를 안전하게 제어하고, 관련된 모든 시스템(회전 모드 변경, 카메라 FOV 조절, 상체 애니메이션 전환)이 동시에 유기적으로 작동하도록 설계한다.
특히 매 프레임 조준 여부를 검사(Tick)하는 비효율적인 방식을 버리고, C++에서 구축한 이벤트 기반(Event-Driven) 아키텍처를 바탕으로 실제 에디터 상에서 어빌리티 간의 상호작용(조준과 달리기의 배타적 실행), 속도 변화, 애니메이션 블렌딩을 GAS 태그와 블루프린트 노드를 통해 유기적으로 제어한다.

## 주요 구조
```text
APlayerCharacter
 ├─ UInputMappingContext & UInputAction (IA_Aim)
 └─ UAbilitySystemComponent (ASC)
     ├─ BP_GA_Aim (Gameplay Ability) : State.Weapon.Aiming 태그 부여 및 Sprint 취소
     │   └─ BP_GE_Aim (Gameplay Effect) : 이동 속도(Attribute) 감속 적용
     └─ OnAimingTagChanged() : 상태 변경 시 회전 모드 강제 전환 및 카메라 줌 타이머 발동

UPlayerAnimInstance
 └─ OnAimingTagChanged() : 상태 변경 시 애니메이션 조준 변수(bIsAiming) 즉시 갱신
     └─ ABP_Player (AnimGraph) : Is Aiming 변수를 활용한 에임 오프셋 블렌딩
```

## 구현 파일
- `Source/TeamProject_Fin/GameCore/Character/PlayerCharacter.h & .cpp`
- `Source/TeamProject_Fin/GameCore/Animation/PlayerAnimInstance.h & .cpp`
- `Content/.../BP_GA_Aim`, `BP_GE_Aim`, `BP_PlayerCharacter`, `ABP_Player`

---

## 주요 구현 내용 및 시스템 역할

### 1. 캐릭터 블루프린트 어빌리티 맵핑 및 트리거 (C++ & BP)
- **입력 바인딩 (Enhanced Input)**: 마우스 우클릭(`IA_Aim`) 입력 발생 시 `StartAim()`, 해제 시 `StopAim()` 함수를 호출한다.
- **안전한 상태 관리**: C++ 내부 변수를 직접 수정하지 않고, 단순히 ASC를 통해 조준 어빌리티를 활성화(TryActivate)하거나 취소(Cancel)한다.
- **BP 세팅**: `BP_PlayerCharacter`의 디테일 패널에서 Aim Action에 `IA_Aim`을 할당하고, 실행될 어빌리티로 `BP_GA_Aim`을 지정했다. 게임 시작 시 ASC가 해당 능력을 소유하도록 Default Abilities에 추가했다.

### 2. 상호 배타적 어빌리티 실행 (조준 vs 달리기) (GAS Tag 제어)
GAS의 태그(Tag) 시스템을 적극 활용하여, 별도의 하드코딩 된 분기문(if문) 없이 어빌리티 간의 우선순위와 취소 로직을 구현했다.
- **`BP_GA_Aim` (조준 어빌리티)**:
  - 실행 시 `State.Weapon.Aiming` 태그를 부여한다.
  - **Cancel Abilities with Tag**: 조준을 시작하면 `State.Movement.Jogging`(달리기) 태그를 가진 능력을 즉시 취소시킨다. 즉, 조준 중에는 달릴 수 없다.
- **`BP_GA_Sprint` (달리기 어빌리티)**:
  - 실행 시 `State.Movement.Jogging` 태그를 부여한다.
  - **Cancel Abilities with Tag**: 달리기를 시작하면 `State.Weapon.Aiming`(조준)과 `State.Movement.Crouching`(앉기) 태그를 가진 능력을 즉시 취소시킨다. 즉, 달리기를 입력하면 조준이 강제로 풀리며 카메라와 회전 모드가 원래대로 복귀한다.

### 3. 조준 중 이동 속도 제어 (Gameplay Effect)
조준 시 캐릭터의 걸음걸이가 느려지는 기획을 게임플레이 이펙트(`BP_GE_Aim`)를 통해 구현했다.
- **속도 모디파이어(Modifier)**: `CharacterAttributeSet.MovementSpeed` 어트리뷰트 값을 200.0으로 덮어쓰기(Override) 한다. 
- 이를 통해 조준 중에는 6번 이슈에서 구현한 `ApplyMovementSpeed()`를 거쳐 캡슐의 `MaxWalkSpeed`가 즉시 갱신되며, 하체 걷기 애니메이션 속도도 자연스럽게 동기화된다.

### 4. 이벤트 기반(Event-Driven) 상태 갱신 시스템 (C++)
- **틱(Tick) 배제 최적화**: ASC 초기화 시점(`PossessedBy`)에서 `RegisterGameplayTagEvent` 기능을 사용해 `State.Weapon.Aiming` 태그의 추가/제거 이벤트를 구독(Subscribe)한다.
- 조준 어빌리티(`BP_GA_Aim`)가 켜지거나 꺼지면서 태그가 변동되는 정확한 찰나의 순간에 단 한 번, `OnAimingTagChanged` 콜백이 실행된다.
- **카메라 및 회전 연동**: 콜백 발생 시 즉각 7번 이슈의 `UpdateCameraZoom()` 타이머 로직을 발동시키고, 강제 회전 모드(`bUseControllerDesiredRotation = true`)로 전환하여 자유 시점 모드를 무효화한다.

### 5. 상체 조준 애니메이션 및 에임 오프셋 블렌딩 (ABP)
C++의 태그 델리게이트를 통해 딜레이 없이 즉각적으로 갱신된 `bIsAiming` 변수를 AnimGraph에서 시각적으로 표현한다.
- **Aim Offset 적용**: 현재 마우스 방향(Aim Yaw, Aim Pitch)에 따라 상체가 마우스를 따라가는 `AO_RifleAim`, `AO_RifleCrouch` 노드를 구성했다.
- **상태 기반 블렌딩**: `Blend Poses by bool` 노드의 Active Value에 `Is Aiming` 변수를 연결했다. 조준 상태(True)일 때는 마우스를 정밀하게 추적하는 에임 오프셋을 상체에 덮어씌우고, 비조준 상태(False)일 때는 일반적인 지향 사격(Hipfire)이나 대기 모션을 캐시된 로코모션과 섞어 출력한다.

---

## 테스트 방법

**상호 취소 테스트 (GAS)**
- 전력 질주(Sprint) 중 우클릭 시, 달리기가 즉시 취소되고 조준 모드로 전환되는지 확인 완료.
- 반대로 조준 중 Shift(달리기) 입력 시, 조준 어빌리티가 강제 취소되며 줌 아웃이 발생하고 달려나가는지 확인 완료.

**속도 동기화 검증**
- 조준 시 이동 속도가 200.0으로 제한되며, 하체 걷기 애니메이션의 재생 속도가 이에 맞춰 자연스럽게 느려지는지 확인 완료.

**자유 시점(Free Look) 무효화 및 에임 오프셋 추적**
- 가만히 서 있을 때(자유 시점 모드) 캐릭터 방향이 마우스와 달라도, 우클릭을 누르면 즉시 카메라 정면 방향으로 부드럽게 회전(`bUseControllerDesiredRotation`)하며 조준 자세를 취하는지 점검.
- 조준 상태 돌입 즉시 애니메이션 블루프린트의 `Is Aiming`이 활성화되며, 딜레이 없이 1프레임 만에 상체 조준 애니메이션으로 블렌딩되어 총구가 정확히 목표를 지향하는지 디버깅 완료.

## 완료 조건 확인
- [x] 조준용 Enhanced Input(`IA_Aim`) 맵핑 및 C++ 바인딩 완료
- [x] 입력 시 GAS 어빌리티(`BP_GA_Aim`) TryActivate/Cancel 흐름 구축
- [x] 어빌리티 간 GAS 태그 기반 Cancel 로직 구축 (Aim vs Sprint 배타적 실행)
- [x] Gameplay Effect(`BP_GE_Aim`)를 통한 이동 속도(Attribute) 오버라이드 적용
- [x] ASC 태그 델리게이트(`RegisterGameplayTagEvent`)를 활용한 틱(Tick) 배제형 이벤트 연동 구현
- [x] 상태 갱신 시 `PlayerCharacter` 회전 모드 강제 전환 및 카메라 줌 타이머 발동 연동
- [x] AnimGraph 내 `Is Aiming` 기반 에임 오프셋 동적 블렌딩 구현

## 남은 작업 또는 확장 방향
-**조준 시 크로스헤어 HUD 생성**: 조준 시 HUD에 숨겨져 있던 크로스헤어가 생성 되어야 한다.
-**조준 시 발사가능**: 조준 시 사격이 활성화 되어야 한다.
