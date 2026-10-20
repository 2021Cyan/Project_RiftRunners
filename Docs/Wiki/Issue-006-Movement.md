# Issue #6 3인칭 캐릭터 이동 구현

## Issue
- **Issue 번호:** #6
- **기능명:** 3인칭 캐릭터 이동 구현 (C++ 로직 및 애니메이션 블루프린트 연동)
- **담당자: seok0317
- **담당 영역:** Character / Movement / Input / Animation Blueprint (ABP) / AnimGraph
- **관련 의존성:** #2 Gameplay Ability System 기본 세팅, #4 GameplayTag 구조 정의, #5 GameplayEffect 기본 구조 구현, 추후 #8 조준 시스템, #9 에임 오프셋과 연계

## 목적
3인칭 캐릭터(플레이어)가 자연스러운 이동(이동, 시점 회전, 앉기 등)을 수행할 수 있도록 C++ 기반의 뼈대와 애니메이션 렌더링 시스템을 구축한다.
C++ 내부에서 수학적으로 정밀하게 계산된 로코모션 데이터(Velocity, Direction, bShouldMove, bIsPivot 등)를 추출하고, 이를 애니메이션 블루프린트(ABP)가 시각적으로 부드럽고 정확하게 렌더링하도록 설계한다. 특히 Start - Loop - Stop - Pivot으로 이어지는 복잡한 이동 상태를 명확하게 분기하고 걷기와 뛰기 애니메이션을 자연스럽게 블렌딩하여 발 꼬임이나 스냅 현상(Popping)이 없는 이동 연출을 구현한다.

## 주요 구조
```text
ABaseCharacter
 ├─ UCharacterMovementComponent (이동 기본 속성 및 콜리전 분리 세팅)
 └─ APlayerCharacter (플레이어 전용 입력 및 동적 회전 제어)
     ├─ UInputMappingContext & UInputAction (Enhanced Input)
     └─ UPlayerAnimInstance (애니메이션 로직 처리 및 데이터 추출)
         └─ ABP_Player (AnimGraph 상태 머신 및 모션 블렌딩)
```

## 구현 파일
- `Source/TeamProject_Fin/GameCore/Character/BaseCharacter.h & .cpp`
- `Source/TeamProject_Fin/GameCore/Character/PlayerCharacter.h & .cpp`
- `Source/TeamProject_Fin/GameCore/Animation/PlayerAnimInstance.h & .cpp`
- `Content/.../ABP_Player.uasset` (애니메이션 블루프린트)

---

## 주요 구현 내용 및 시스템 역할

### 1. 캐릭터 콜리전 분리 및 기본 세팅 (C++ / ABaseCharacter)
- **콜리전 최적화**: 캡슐 컴포넌트는 오직 지형 충돌과 이동 간 겹침 방지(Pawn) 용도로만 사용하며, `Camera` 및 전투용 트레이스 채널(`PlayerAttack`, `EnemyAttack`)을 무시(Ignore)하도록 세팅하여 이동 중 걸림 현상을 방지했다.
- **속도 동기화**: `ApplyMovementSpeed()` 함수를 통해 GAS Attribute의 이동 속도가 변경될 때 `MaxWalkSpeed`와 앉기 속도(`MaxWalkSpeedCrouched`)를 동기화하여 시스템 일관성을 유지한다.

### 2. 동적 회전 제어 및 상태 동기화 (C++ / APlayerCharacter)
- **동적 회전 모드(Dynamic Rotation Mode)**: 
  - **이동 입력(WASD) 중**: `bUseControllerDesiredRotation = true`로 설정하여 캐릭터가 즉시 홱 도는 팝핑 현상 없이 카메라가 바라보는 방향으로 부드럽게 스트레이프(Strafe) 이동을 수행한다.
  - **대기 중 (자유 시점)**: `bOrientRotationToMovement = true`로 설정하여, 유저가 마우스로 주변을 둘러보아도 캐릭터의 몸통이 따라 돌아가지 않는 자유 시점을 제공한다.
- **안전한 상태 동기화**: 엔진이 물리적으로 앉기를 처리하는 순정 이벤트(`OnStartCrouch`, `OnEndCrouch`)에서 회전 모드 동기화 로직을 호출하여, 프레임 오차에 의한 상태 머신 꼬임을 완벽하게 방지했다.

### 3. 수학적 로코모션 데이터 추출 (C++ / UPlayerAnimInstance)
- **이동 데이터 추출**: `Velocity`, `Acceleration`을 바탕으로 `GroundSpeed`, `bHasAcceleration`, `bShouldMove` 등을 틱 단위로 계산.
- **정밀한 피벗(Pivot) 알고리즘**: 이동 중 급격한 방향 전환 시 `Acceleration`과 `Velocity`의 내적(DotProduct)이 일정 수치 이하로 떨어지는 순간 피벗 상태(`bIsPivot`)를 발동시킨다. 가속도를 로컬 공간으로 역변환 후 `FMath::Atan2`로 정확한 로컬 Yaw 각도를 추출해 방향(`CurrentPivotDirection`)을 4방향으로 정밀 분류한다.
- **정지(Stop) 방향 유지**: 가속도와 속도의 방향이 일치하는(내적이 양수인) 정상 주행 상태일 때만 `Direction`을 `SavedMovementDirection`에 갱신하여, 정지 시 발이 꼬이거나 허리가 틀어지는 현상을 방지했다.

### 4. 상태 머신 (State Machine) 분기 구조 (ABP / AnimGraph)
- 캐릭터의 서 있는 상태(Stand)와 앉은 상태(Crouch)를 분리하여 동일한 흐름의 로코모션 사이클을 구축했다.
- **흐름**: Idle ↔ Start ↔ Loop ↔ Stop / Loop ↔ Pivot
- C++에서 추출한 `bShouldMove`를 활용해 Idle에서 Start로 진입하며, 내적으로 계산된 `bIsPivot` 변수를 통해 방향 전환 시 즉각적으로 Pivot 상태로 전이되도록 트랜지션 룰을 구성했다.
- Stop 상태의 애니메이션 재생이 완료되거나 특정 조건이 충족되면 다시 Idle로 복귀하는 구조를 통해 자연스러운 제동을 연출한다.

### 5. 이동 시작 (Start) 모션 블렌딩 (ABP)
- 단순히 하나의 애니메이션을 재생하는 것이 아니라 상태와 방향에 맞춰 정교하게 섞이도록 구현했다.
- `BS_Jog_Start`와 `BS_Walk_Start` 두 개의 블렌드스페이스를 사용해 C++에서 넘어온 `Direction` 값에 따라 올바른 발진 방향 모션을 결정한다.
- `Is Jogging` 불리언 변수를 `Blend Poses by bool` 노드의 Active Value로 사용하여, 걷기 시작과 뛰기 시작 모션을 부드럽게(Blend Time 0.1) 전환한다.

### 6. 4방향 정밀 피벗 (Pivot) 시스템 적용 (ABP)
- 단순히 180도 회전 애니메이션만 재생하는 것을 넘어, 전후좌우 각기 다른 4방향 피벗을 지원한다.
- C++의 수학적 연산으로 도출해낸 `Current Pivot Direction`(Enum) 값을 `Blend Poses (EPivotDirection)` 노드에 연결하여 Forward, Backward, Left, Right 모션 시퀀스를 정확하게 분기한다.
- 피벗 동작 시에도 걷기와 뛰기(`Is Jogging`) 상태를 구분하여 각기 다른 웨이트의 피벗 시퀀스를 재생, 발 미끄러짐(Foot Sliding)을 최소화했다.

### 7. 상/하체 분리 및 에임 오프셋(AO) 뼈대 연동 (ABP)
- 이동 로코모션을 하체에 유지한 채 상체에서 무기를 다루거나 조준할 수 있도록 캐시(Cache)와 레이어드 블렌드를 적용했다.
- 완성된 하체 이동 로직을 `Cache_Locomotion`으로 저장한다.
- 상체에는 상태(`Is Aiming`, `Is Crouching`)에 따라 각각 다른 Aim Offset(`AO_RifleAim`, `AO_RifleCrouch`, `AO_RifleHip`)을 적용한다.
- `Layered blend per bone` 노드를 사용하여 조준 상태나 이동 상태(Jogging)에 따라 척추(Spine)를 기준으로 상/하체 애니메이션이 어색함 없이 섞이도록 웨이트를 조절했다.

---

## 테스트 방법

**에디터 및 블루프린트 설정**
- `IMC_Default`, `IA_Move` 등을 `BP_PlayerCharacter`에 할당하고, 애니메이션 블루프린트(`ABP_Player`)에서 C++ 변수들을 트랜지션 룰에 연결 확인.

**PIE 동작 확인**
- **조작 반응성 및 블렌딩**: WASD 입력 시 Idle에서 즉시 Loop로 튀지 않고 Start 애니메이션을 거쳐 부드럽게 가속하며 카메라 방향으로 이동함.
- **자유 시점**: 입력을 멈췄을 때 마우스 룩이 캐릭터 몸통 회전에 영향을 주지 않고 자연스러운 시점 궤도를 그림.
- **피벗(Pivot)**: 달리는 중 반대/측면 방향 전환 시, 즉각적으로 4방향 Pivot 모션이 출력되며 발 미끄러짐이 방지됨.
- **앉기(Crouch)**: 앉은 상태에서도 이동 입력 여부에 따라 올바른 회전 모드가 유지됨.

**애니메이션 디버그 및 빌드 검증**
- Show Debug Animation을 통해 State Machine이 Start -> Loop -> Stop 흐름을 정확한 프레임에 맞춰 전이하는지 가중치 검증 완료.
- Unreal Engine 5.8 Development Editor 빌드 성공 (Result: Succeeded).

## 완료 조건 확인
- [x] 이동/시점 회전용 Enhanced Input 세팅 및 캡슐 속성 튜닝
- [x] 동적 회전 모드(Strafe vs FreeLook) 분기 및 `bUseControllerDesiredRotation` 보간 적용
- [x] 애니메이션 데이터(속도, 방향) 틱 추출 로직 및 역행렬/Atan2 기반 피벗 감지 구현
- [x] ABP 상태 머신 (Idle-Start-Loop-Stop-Pivot) 구축 및 트랜지션 룰 연동
- [x] Start 모션 및 4방향 Pivot 시퀀스의 조깅/걷기 상태별 동적 블렌딩 구현
- [x] 하체 이동 로직 캐시(Cache) 분리 및 상체 에임 오프셋 레이어드 블렌딩 뼈대 구현

## 남은 작업 또는 확장 방향
- **GAS 확장**:  `GA_Dodge`(회피)와 피격 애니메이션과 연동하여 이동 속도 변화가 즉시 애니메이션 속도 및 블렌딩에 매끄럽게 반영되도록 설계 연장.