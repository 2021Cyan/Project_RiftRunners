# Issue #18 장전 시스템 및 사격 상태 제어 개선

## 구현 목적

GAS 기반 장전 Ability를 추가하고, 장전 애니메이션의 특정 시점에서 실제 탄약이 갱신되도록 몽타주 기반 장전 흐름을 구현한다.

장전 중에는 사격과 왼손 IK를 제한하고, 장전 종료 후에는 왼손이 무기 Grip 위치로 자연스럽게 복귀하도록 처리한다.

또한 Sprint / Dodge와 같은 이동 상태 중에는 사격 Ability가 실행되지 않도록 GameplayTag 기반 사격 제한을 추가한다.

## 주요 구현 내용

### 장전 흐름

1. 플레이어 입력에서 `GA_Reload`를 활성화한다.
2. `GA_Reload`는 현재 장착된 무기의 `CanReload()`를 확인한다.
3. 장전 가능 상태라면 `ReloadMontage`를 재생한다.
4. Ability가 실행되는 동안 `State.Weapon.Reloading` 태그를 유지한다.
5. 장전 몽타주의 탄창 삽입 시점에서 GameplayEvent를 발생시킨다.
6. `GA_Reload`가 `Event.Weapon.ReloadCommit`을 수신하면 `UWeaponComponent::ApplyReload()`를 호출한다.
7. 실제 탄약 갱신이 완료되면 `Event.Weapon.Reloaded` 이벤트를 발생시킨다.
8. 몽타주가 종료되면 `GA_Reload`를 종료하고 장전 상태 태그를 제거한다.

### 몽타주 기반 장전 완료 처리

장전 시간 기반 Timer를 사용하지 않고, 실제 애니메이션의 탄창 삽입 프레임을 기준으로 장전을 완료한다.

몽타주 내부의 특정 프레임에 GameplayEvent Notify를 배치한다.

```text
Reload Montage
    ↓
AnimNotify_SendGameplayEvent
    ↓
Event.Weapon.ReloadCommit
    ↓
GA_Reload::OnReloadCommit
    ↓
WeaponComponent::ApplyReload
```

이를 통해 장전 애니메이션 길이가 변경되어도 실제 탄약 갱신 시점이 애니메이션과 분리되지 않는다.

장전 애니메이션이 `ReloadCommit` 이전에 취소되면 탄약은 갱신되지 않는다.

`ReloadCommit` 이후에 몽타주가 취소되면 이미 적용된 탄약은 유지된다.

### 공용 GameplayEvent AnimNotify

장전 전용 Notify 대신 범용 `UAnimNotify_SendGameplayEvent`를 사용한다.

Notify는 에디터에서 지정된 `FGameplayTag`를 현재 Animation Owner Actor에게 전달한다.

```text
AnimNotify_SendGameplayEvent
    ↓
EventTag
    ↓
SendGameplayEventToActor
```

장전에서는 다음 태그를 사용한다.

```text
Event.Weapon.ReloadCommit
```

동일한 Notify는 이후 다른 Montage 이벤트에서도 재사용할 수 있다.

예시:

```text
Event.Weapon.ReloadCommit
Event.Skill.Cast
```

### 장전 상태 GameplayTag

장전 관련 상태와 이벤트를 다음 GameplayTag로 관리한다.

```text
Ability.Weapon.Reload
State.Weapon.Reloading
Event.Weapon.ReloadCommit
Event.Weapon.Reloaded
```

`State.Weapon.Reloading`은 `GA_Reload`이 활성화된 동안 유지된다.

중복 장전을 방지하기 위해 `GA_Reload`은 동일한 상태 태그를 Activation Block 조건으로 사용한다.

### 장전 중 사격 차단

`GA_Fire`에 다음 상태 태그를 Activation Block 조건으로 추가한다.

```text
State.Weapon.Reloading
```

따라서 장전 Ability가 실행 중일 때는 새로운 사격 Ability를 활성화할 수 없다.

### Sprint / Dodge 중 사격 차단

이동 상태 중 사격을 제한하기 위해 다음 태그를 사용한다.

```text
State.Movement.Sprinting
State.Movement.Dodging
```

`GA_Fire`의 `ActivationBlockedTags`에 두 태그를 추가한다.

```text
State.Weapon.Reloading
State.Movement.Sprinting
State.Movement.Dodging
```

따라서 다음 상태에서는 사격 Ability가 활성화되지 않는다.

```text
장전 중
Sprint 중
Dodge 중
```

Sprint / Dodge 시작 시 이미 실행 중인 `Ability.Weapon.Fire`도 취소하도록 처리한다.

이를 통해 연사 중 이동 Ability를 실행했을 때 기존 사격 Timer가 계속 동작하는 문제를 방지한다.

### Sprint 상태 태그

Sprint Ability가 활성화된 동안 다음 상태 태그를 사용한다.

```text
State.Movement.Sprinting
```
`GA_Sprint`의 `ActivationOwnedTags`를 이용해 Ability 생명주기와 상태 태그를 연결하여 

Sprint가 종료되면 상태 태그도 함께 제거된다.

### Dodge 상태 태그

기존 Dodge 로직에서 직접 추가/제거하던:

```text
State.Movement.Dodging
```

태그를 사격 차단 조건으로 사용한다.

Dodge Ability가 활성화된 동안 사격을 제한하고, Dodge 종료 후 다시 사격할 수 있도록 처리한다.

## 왼손 무기 Grip IK

### HandSocket 기반 왼손 위치 보정

무기 `BarrelMesh`에 왼손이 잡아야 하는 위치를 나타내는 `HandSocket`을 추가한다.

`UPlayerAnimInstance`는 현재 장착된 무기의 `HandSocket` Transform을 매 프레임 조회한다.

```text
Weapon
└─ BarrelMesh
   └─ HandSocket
```

왼손 IK 기준 Bone으로 캐릭터 Skeleton의 기존 `ik_hand_gun` Bone을 사용한다.

```text
ik_hand_root
└─ ik_hand_gun
   ├─ ik_hand_l
   └─ ik_hand_r
```

무기의 `HandSocket` World Transform을 캐릭터의 `ik_hand_gun` World Transform 기준 Relative Transform으로 변환한다.

```cpp
LeftHandIKTransform =
    HandSocketWorldTransform.GetRelativeTransform(
        GunIKWorldTransform
    );
```

계산된 Transform은 AnimBP의 FABRIK Effector Transform으로 사용한다.

### FABRIK 설정

AnimGraph 최종 Pose 직전에 FABRIK를 배치한다.

```text
Tip Bone
= hand_l

Root Bone
= upperarm_l

Effector Transform Space
= Bone Space

Effector Target
= ik_hand_gun
```

`LeftHandIKTransform`을 FABRIK Effector Transform에 연결한다.

이를 통해 Aim Offset, 이동 애니메이션, UpperBody Montage 등이 적용된 이후에도 왼손이 무기의 핸드가드 위치를 따라간다.

### 장전 중 왼손 IK 해제

장전 중에는 왼손이 탄창을 교체해야 하므로 무기 HandSocket에 고정되면 안 된다.

`State.Weapon.Reloading` 태그를 확인해 FABRIK Alpha를 제어한다.

```text
평상시
LeftHandIKAlpha = 1

장전 중
LeftHandIKAlpha = 0
```

IK Transform 계산 자체는 계속 유지하고, FABRIK 영향만 Alpha로 제거한다.

### 장전 종료 시 IK 복귀 보간

장전 종료 직후 Alpha를 `0 → 1`로 즉시 변경하면 왼손이 무기 HandSocket 위치로 순간적으로 Snap되는 현상이 발생한다.

이를 방지하기 위해 `FMath::FInterpTo`를 사용해 FABRIK Alpha를 부드럽게 보간한다.

## UpperBody Montage / Aim Offset 처리

### 이동 중 장전

장전 Montage는 캐릭터 전체 Pose를 대체하지 않고 `UpperBody` Slot으로 재생한다.

AnimBP에서는 `Layered Blend Per Bone`을 이용해 하체 Locomotion은 유지하고 상체만 Montage를 적용한다.

```text
Base Pose
= Locomotion

Blend Pose
= UpperBody Montage

Branch Filter
= spine_01 이상
```

이를 통해 걷기 / 달리기 중에도 하체 이동 애니메이션을 유지하면서 장전할 수 있다.

### Aim Offset과 UpperBody Montage 적용 순서

UpperBody Montage가 Aim Offset 이후에 적용되면 Montage의 기본 정면 포즈가 기존 조준 방향을 덮어쓸 수 있다.

예를 들어 캐릭터가 왼쪽을 Aim Offset으로 바라보고 있는 상태에서 Reload Montage를 재생하면 상체가 다시 정면을 바라보는 문제가 발생한다.

이를 방지하기 위해 UpperBody Montage 결과를 Aim Offset의 Base Pose로 사용한다.

```text
Locomotion
    ↓
UpperBody Montage
    ↓
Aim Offset
    ↓
Layered Blend Per Bone
    ↓
Left Hand FABRIK
```

따라서 Montage가 재생 중이더라도 현재 Aim Yaw / Pitch 방향이 최종 상체 Pose에 유지된다.

Aim / Crouch / Hip 상태별 AimOffset을 사용하는 경우 각 AimOffset의 Base Pose에 UpperBody Montage가 적용된 Pose를 전달한다.

## 관련 구현

* `Source/TeamProject_Fin/GameCore/Ability/GA_Fire.h`
* `Source/TeamProject_Fin/GameCore/Ability/GA_Fire.cpp`
* `Source/TeamProject_Fin/GameCore/Ability/GA_Reload.h`
* `Source/TeamProject_Fin/GameCore/Ability/GA_Reload.cpp`
* `Source/TeamProject_Fin/GameCore/Ability/GA_Sprint.h`
* `Source/TeamProject_Fin/GameCore/Ability/GA_Sprint.cpp`
* `Source/TeamProject_Fin/GameCore/Ability/GA_Dodge.h`
* `Source/TeamProject_Fin/GameCore/Ability/GA_Dodge.cpp`
* `Source/TeamProject_Fin/GameCore/Tags/GameplayTagSystem.h`
* `Source/TeamProject_Fin/GameCore/Tags/GameplayTagSystem.cpp`
* `Source/TeamProject_Fin/GameCore/Animation/PlayerAnimInstance.h`
* `Source/TeamProject_Fin/GameCore/Animation/PlayerAnimInstance.cpp`
* `Source/TeamProject_Fin/GameCore/Animation/Animnotify_SendGameplayEvent.h`
* `Source/TeamProject_Fin/GameCore/Animation/Animnotify_SendGameplayEvent.cpp`
* `Source/TeamProject_Fin/GameCore/Weapon/Core/WeaponComponent.h`
* `Source/TeamProject_Fin/GameCore/Weapon/Core/WeaponComponent.cpp`
* `Source/TeamProject_Fin/GameCore/Weapon/Core/WeaponBase.h`
* `Source/TeamProject_Fin/GameCore/Weapon/Core/WeaponBase.cpp`

## 사용 방법

### 장전 Ability 설정

1. `GA_Reload` 기반 GameplayAbility Blueprint를 만든다.
2. `ReloadMontage`에 사용할 장전 Montage를 지정한다.
3. 플레이어의 기본 Ability 목록에 Reload Ability를 등록한다.
4. Reload Input Action을 PlayerCharacter에 연결한다.
5. 입력 시 `TryActivateAbilityByClass`로 `GA_Reload`을 실행한다.

### 장전 Montage 설정

1. Rifle Reload Animation으로 Montage를 생성한다.
2. Montage Slot을 `UpperBody`로 설정한다.
3. 탄창이 실제로 총에 삽입되는 프레임에 `AnimNotify_SendGameplayEvent`를 배치한다.
4. Notify의 EventTag를 다음 값으로 설정한다.

```text
Event.Weapon.ReloadCommit
```

### UpperBody Slot 설정

Skeleton Anim Slot Manager에 다음 Slot을 추가한다.

```text
DefaultGroup
└─ UpperBody
```

AnimBP의 Slot 노드와 Reload Montage의 Slot Track이 동일한 `UpperBody` Slot을 사용해야 한다.

### HandSocket 설정

각 무기의 `BarrelMesh`에 다음 Socket을 만든다.

```text
HandSocket
```

Socket 위치는 왼손 손바닥이 실제로 핸드가드를 잡아야 하는 위치에 맞춘다.

손목 회전이 필요한 경우 Socket Rotation도 손바닥 방향에 맞게 조정한다.

### FABRIK 설정

AnimBP에서 FABRIK를 최종 Pose 근처에 배치한다.

```text
Tip Bone = hand_l
Root Bone = upperarm_l
Effector Transform Space = Bone Space
Effector Target = ik_hand_gun
Effector Transform = LeftHandIKTransform
Alpha = LeftHandIKAlpha
```

### 사격 제한 설정

`GA_Fire`는 다음 상태 태그에서 활성화를 차단한다.

```text
State.Weapon.Reloading
State.Movement.Sprinting
State.Movement.Dodging
```

Sprint / Dodge 시작 시 이미 실행 중인 `Ability.Weapon.Fire`도 취소한다.

## 테스트 및 확인 내용

- [x] 탄창이 가득 찬 상태에서는 Reload Ability가 실행되지 않는지 확인
- [x] 탄창이 비어 있거나 일부 소모된 상태에서 Reload 입력 시 Montage가 재생되는지 확인
- [x] Montage의 `ReloadCommit` Notify 이전에는 탄약이 변경되지 않는지 확인
- [ ] `ReloadCommit` 시점에 `CurrentAmmo`가 기존 `MagazineSize`까지 채워지는지 확인 (예비 탄약 무제한 전환 후 PIE 재검증 필요)
- [x] 장전 중 다시 Reload 입력을 눌러도 중복 Ability가 실행되지 않는지 확인
- [x] 장전/Sprint/Dodge 도중 Fire 입력이 차단되는지 확인
- [x] 왼손이 무기의 `HandSocket` 위치를 정상적으로 따라가는지 확인
- [x] 장전 시작 시 왼손 IK가 해제되어 Reload Montage의 왼손 움직임이 정상적으로 재생되는지 확인

## 남은 작업 또는 확장 방향

* 장전 중 사격 입력 시 장전 취소 후 즉시 사격하는 인터럽트 처리
* 탄약 HUD와 `Event.Weapon.Reloaded` 연동

