# Issue #7 TPS 카메라 구현

## Issue
- **Issue 번호:** #7
- **기능명:** 3인칭(TPS) 숄더뷰 카메라 시스템 구현
- **담당자: seok0317
- **담당 영역:** Camera / PlayerCharacter / Optimization
- **관련 의존성:** #6 3인칭 캐릭터 이동 구현, #8 조준 시스템(FOV 연동)

## 목적
플레이어 캐릭터에 3인칭 슈팅(TPS)에 최적화된 카메라를 부착하고, 줌 인/아웃(조준) 및 앉기(Crouch) 등의 상태 변화 시 카메라가 어지럽게 튀는 현상(Snapping)을 완벽히 방지한다.
모든 카메라의 부드러운 보간(Interpolation) 처리는 성능 최적화를 위해 엔진의 무거운 매 프레임 연산(`Tick`)을 배제하고, 타이머(Timer) 기반의 독립적인 연산으로 대체하여 설계한다.

## 주요 구조
```text
APlayerCharacter
 ├─ USpringArmComponent (CameraBoom) : 카메라 거리 및 충돌 처리
 │   └─ UCameraComponent (FollowCamera) : 실제 시각 및 FOV 담당
 ├─ FTimerHandle ZoomTimerHandle : 조준 줌(FOV) 보간용 타이머
 └─ FTimerHandle CrouchCameraTimerHandle : 앉기 시 높이(Z축) 보간용 타이머
```

## 구현 파일
- `Source/TeamProject_Fin/GameCore/Character/PlayerCharacter.h`
- `Source/TeamProject_Fin/GameCore/Character/PlayerCharacter.cpp`

## 클래스 역할 및 주요 로직

### APlayerCharacter (카메라 파트)
캐릭터의 루트(캡슐) 컴포넌트에 스프링암과 카메라를 부착하여 숄더뷰 시점을 제공한다.

1. **카메라 부착 및 기본 설정**
   - `CameraBoom->bUsePawnControlRotation = true`로 설정하여 마우스 입력(Controller Rotation)에 따라 카메라가 상하좌우로 회전하도록 세팅한다.
   - 캐릭터 모델이 카메라에 너무 가깝게 렌더링되지 않도록 `TargetArmLength` 및 `SocketOffset`을 설정할 수 있는 기반을 만든다.

2. **조준(Aim) 줌 인/아웃 보간 (Tick-Free)**
   - GAS 태그 콜백(`OnAimingTagChanged`)을 통해 조준 상태 변화를 감지한다.
   - 상태가 변하면 `ZoomTimerHandle`을 실행하여 초당 약 60프레임(`0.016f`) 간격으로 `UpdateCameraZoom()` 함수를 호출한다.
   - `FMath::FInterpTo`를 사용해 현재 FOV를 목표 FOV(`AimFOV` 또는 `DefaultFOV`)로 부드럽게 보간하며, 목표치에 근접(`FMath::IsNearlyEqual`)하면 즉시 타이머를 해제(ClearTimer)하여 불필요한 연산 리소스를 절약한다.

3. **앉기(Crouch) 카메라 덜컹거림(Jitter) 완벽 상쇄**
   - **문제점**: 언리얼 엔진 기본 `Crouch()` 호출 시 캡슐 콜리전 높이가 절반으로 깎이며 루트 컴포넌트의 위치 기준점이 바닥으로 강제 이동한다. 이때 루트에 매달린 카메라도 순간이동하듯 뚝 떨어져 플레이어에게 멀미를 유발하는 현상이 발생함.
   - **해결 방안 (순간 오프셋 상쇄)**: 
     - 엔진의 `OnStartCrouch`를 오버라이드하여 캡슐이 줄어드는 정확한 찰나의 수치(`HalfHeightAdjust`)만큼 스프링암의 `TargetOffset.Z`를 반대로 순간적으로 끌어올린다. (이를 통해 캐릭터가 앉는 프레임에 카메라가 0.1mm도 흔들리지 않고 고정됨)
     - 직후 `CrouchCameraTimerHandle`을 작동시켜, 유저가 설정한 카메라 하강 수치(`CrouchCameraDropAmount`, 기본 45)만큼만 부드럽게 보간하여 스르륵 내려가게 만든다.
   - **복구 로직**: `OnEndCrouch`에서는 반대로 위로 솟구치는 캡슐만큼 스프링암을 순간적으로 내린 뒤, 원래 높이(`0.0f`)로 부드럽게 복구한다.

## 에디터(블루프린트) 세부 파라미터 설정

### 1. Zoom (조준 시야각 설정)
조준(Aiming) 상태 진입 및 해제 시 타이머를 통해 보간될 카메라의 시야각(FOV)과 보간 속도를 결정한다.
- **Default FOV (90.0)**: 비조준(기본 이동 및 대기) 상태일 때의 기본 카메라 시야각. 시야를 넓게 확보하여 주변 환경 탐색에 유리하게 설정되었다.
- **Aim FOV (60.0)**: 조준 상태일 때 목표로 하는 시야각. 화면을 줌 인하여 타겟팅에 집중할 수 있도록 FOV를 좁힌다.
- **Zoom Interp Speed (15.0)**: 시야각 전환 시의 보간 속도. 값이 클수록 목표 FOV에 도달하는 시간이 짧아져 조준 반응성이 빠르고 날카로워진다.

### 2. Crouch (앉기 카메라 보간 설정)
C++에서 `OnStartCrouch` 시 발생하는 캡슐의 순간적인 축소로 인한 카메라 덜컹거림(Jitter)을 방지한 후, 스르륵 내려가는 연출의 깊이와 속도를 제어한다.
- **Crouch Camera Drop Amount (14.0)**: 앉았을 때 카메라가 최종적으로 하강하는 Z축 깊이 값. 캡슐의 절반 높이만큼 무조건 떨어지는 순정 상태와 달리, 사용자가 어지러움을 느끼지 않는 최적의 수치인 14.0으로 하강 폭을 제한했다.
- **Crouch Camera Interp Speed (6.0)**: 카메라가 목표 Z축으로 하강하거나 상승하는 보간 속도. 줌 인/아웃 속도(15.0)에 비해 상대적으로 낮게 설정(6.0)하여, 앉기 동작 시 카메라가 묵직하고 부드럽게 이동하도록 연출한다.

## 테스트 방법
**에디터 및 블루프린트 설정**
- `BP_PlayerCharacter` 디테일 패널에서 `Camera | Crouch` 카테고리의 `CrouchCameraDropAmount`(하강 깊이) 및 `CrouchCameraInterpSpeed`(보간 속도) 수치 튜닝.
- 조준 시 사용할 `AimFOV` 값 설정.

**PIE 동작 확인**
- **조준**: 마우스 우클릭 시 FOV가 끊김 없이 부드럽게 줌 인/아웃 되는지 확인. 줌이 완료된 후 타이머가 정상적으로 멈춰 성능을 낭비하지 않는지 확인.
- **앉기 보간**: 걷거나 서 있는 상태에서 앉기 키 입력 시, 화면이 뚝 떨어지지 않고 부드럽게 하강 및 상승하는지 확인.

## 완료 조건 확인
- [x] 스프링암 및 카메라 컴포넌트 C++ 초기 세팅 완료
- [x] Tick을 배제한 Timer 기반 카메라 FOV 조준 연산 최적화 구현
- [x] `OnStartCrouch`, `OnEndCrouch` 오버라이드를 통한 캡슐 축소/확장 시각적 상쇄(Jitter 방지) 알고리즘 구현
- [x] 앉기 카메라 Z축 오프셋 타이머 보간 처리
- [x] 에디터(블루프린트)에서 카메라 드롭 깊이와 속도를 조절할 수 있도록 UPROPERTY 프로퍼티 개방

## 남은 작업 또는 확장 방향
-**구르기, 피격 연출**: `GA_Dodge`(회피)와 피격 애니메이션 때 자연스러운 카메라 연출 추가 