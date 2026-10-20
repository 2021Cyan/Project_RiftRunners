# Issue #19 반동 시스템 구현

## 구현 목적

총기 발사 시 로컬 플레이어의 카메라에 무기별 반동을 적용하고, 시간이 지나면 발사 전 방향으로 자연스럽게 회복되도록 구현한다.

실제 탄도에는 무기별 Spread를 적용하며, 조준 상태에서는 반동과 Spread가 각각 설정된 배율만큼 감소하도록 구성한다.

## 주요 구현 내용

### 반동 적용 흐름

1. 플레이어 입력으로 `GA_Fire`를 활성화한다.
2. 서버는 `UWeaponComponent::ExecuteFire()`를 통해 실제 발사를 처리한다.
3. 발사가 성립하면 현재 무기의 `FWeaponRuntimeStats::Recoil` 설정을 조회한다.
4. 조준 중이면 `Recoil.AimingMultiplier`를 Pitch에 적용한다.
5. `APlayerCharacter::ApplyWeaponRecoil()`이 로컬 플레이어의 Controller Rotation에 Pitch/Yaw 반동을 적용한다.
6. 적용한 반동의 반대 값을 누적하고 Tick에서 부드럽게 회복한다.

카메라 반동은 실제 화면을 소유한 로컬 플레이어에게만 적용한다. 

### 무기별 반동 데이터

반동 설정은 `FWeaponRuntimeStats` 내부의 `FWeaponRecoilStats`로 관리한다.

- `Pitch`: 발사 1회당 위쪽 카메라 반동 크기
- `YawRatio`: Pitch 대비 좌우 무작위 반동 범위
- `AimingMultiplier`: 조준 중 반동 배율

```text
Yaw 범위 = -Pitch * YawRatio ~ Pitch * YawRatio
조준 중 Pitch = Pitch * AimingMultiplier
```

각 무기의 `WeaponDataAsset.RuntimeStats.Recoil` 값을 다르게 설정하면 라이플, 저격총, 유탄발사기마다 서로 다른 반동을 사용할 수 있다.

### 반동 계산 Library

반동 수치 계산은 `UWeaponRecoilLibrary`에 분리한다.

- `CalculateRecoilKick()`: Pitch와 YawRatio로 1회 반동값 계산
- `CalculateRecoveryStep()`: 현재 남은 회복량과 속도로 이번 프레임 회복량 계산
- `SnapSmallRecovery()`: 허용 오차보다 작은 잔여값을 0으로 정리

`UBlueprintFunctionLibrary`로 구현되어 C++과 Blueprint에서 동일한 계산 함수를 사용할 수 있다.

### 로컬 카메라 반동

Pitch 반동은 음수 입력으로 카메라를 위로 올리고, Yaw 반동은 좌우 범위에서 무작위로 선택한다.

```cpp
AddControllerPitchInput(PitchKick);
AddControllerYawInput(YawKick);
```

발사 Ability는 로컬 예측 상태에서도 반동을 즉시 적용한다. 따라서 클라이언트가 발사 입력을 보낸 뒤 서버 응답을 기다리느라 카메라 반동이 늦게 보이지 않는다.

서버에서 로컬 적용이 필요한 경우에는 `ClientApplyWeaponRecoil` RPC를 통해 소유 클라이언트에 반동값을 전달한다.

### 반동 회복

반동 적용 시 Pitch/Yaw 반대 방향의 회복량을 누적한다.

```text
PendingRecovery += -RecoilKick
```

로컬 플레이어의 Tick에서 `RecoilRecoverySpeed`와 `DeltaTime`을 사용해 남은 회복량의 일부를 Controller Rotation에 되돌린다.

```text
RecoveryStep = PendingRecovery * Clamp(RecoverySpeed * DeltaTime, 0, 1)
PendingRecovery -= RecoveryStep
```

회복값이 충분히 작아지면 Snap 처리하여 미세한 회복 연산이 계속 남지 않도록 한다.

### Spread 적용

실제 탄도 분산은 `FWeaponRuntimeStats::Spread`로 관리한다.

- `Angle`: 비조준 상태의 원뿔 반각
- `AimingMultiplier`: 조준 중 Spread 배율

`AWeaponBase::GetFireDirection()`은 카메라 기준 조준 방향을 계산한 뒤 `FMath::VRandCone`으로 Spread를 적용한다.

```text
비조준 Spread = Spread.Angle
조준 Spread = Spread.Angle * Spread.AimingMultiplier
```

Spread는 서버의 실제 HitScan 및 Projectile 발사 방향에 적용되므로 피격 판정과 투사체 이동이 동일한 최종 방향을 사용한다.

### 조준 상태 연동

반동과 Spread는 `State.Weapon.Aiming` GameplayTag를 기준으로 각각 감소한다.

- 카메라 반동: `Recoil.AimingMultiplier`
- 실제 탄도 분산: `Spread.AimingMultiplier`

두 배율은 서로 독립적이다. 예를 들어 저격총은 두 값을 매우 낮게 설정해 조준 시 정밀도를 크게 높일 수 있고, 라이플은 완전히 고정되지 않도록 더 높은 값을 사용할 수 있다.

## 관련 구현

* `Source/TeamProject_Fin/GameCore/Ability/GA_Fire.h`
* `Source/TeamProject_Fin/GameCore/Ability/GA_Fire.cpp`
* `Source/TeamProject_Fin/GameCore/Character/PlayerCharacter.h`
* `Source/TeamProject_Fin/GameCore/Character/PlayerCharacter.cpp`
* `Source/TeamProject_Fin/GameCore/Weapon/Core/WeaponBase.h`
* `Source/TeamProject_Fin/GameCore/Weapon/Core/WeaponBase.cpp`
* `Source/TeamProject_Fin/GameCore/Weapon/Data/WeaponTypes.h`
* `Source/TeamProject_Fin/GameCore/Weapon/Data/WeaponDataAsset.h`
* `Source/TeamProject_Fin/GameCore/Weapon/Libraries/WeaponRecoilLibrary.h`
* `Source/TeamProject_Fin/GameCore/Weapon/Libraries/WeaponRecoilLibrary.cpp`
* `Source/TeamProject_Fin/GameCore/Tags/GameplayTagSystem.h`
* `Source/TeamProject_Fin/GameCore/Tags/GameplayTagSystem.cpp`

## 사용 방법

### 무기별 반동 설정

`WeaponDataAsset.RuntimeStats.Recoil`에서 다음 값을 설정한다.

```text
Pitch
YawRatio
AimingMultiplier
```

권장 방향:

- 라이플: 중간 Pitch, 중간 YawRatio, 조준 시 일부 감소
- 저격총: 큰 Pitch, 낮은 YawRatio, 조준 시 크게 감소
- 유탄발사기: 큰 Pitch, 낮거나 중간 YawRatio

### 무기별 Spread 설정

`WeaponDataAsset.RuntimeStats.Spread`에서 다음 값을 설정한다.

```text
Angle
AimingMultiplier
```

저격총은 조준 시 높은 정밀도가 필요하므로 `AimingMultiplier`를 매우 낮게 설정한다.

### 반동 회복 속도 설정

플레이어 캐릭터의 다음 값을 조정한다.

```text
Camera > Recoil > RecoilRecoverySpeed
```

값이 높을수록 빠르게 원래 조준 방향으로 돌아오고, 값이 낮을수록 반동이 오래 남는다.

## 테스트 및 확인 내용

- [x] 발사 성공 시 로컬 플레이어 카메라에 Pitch/Yaw 반동이 적용되는지 확인
- [x] 발사하지 않으면 누적 반동이 자연스럽게 회복되는지 확인
- [x] 조준 중 반동이 `Recoil.AimingMultiplier`에 따라 감소하는지 확인
- [x] 조준 중 Spread가 `Spread.AimingMultiplier`에 따라 감소하는지 확인
- [x] 무기 RuntimeStats를 다르게 설정했을 때 무기별 반동과 Spread 차이가 발생하는지 확인
- [x] 클라이언트 발사 시 소유 클라이언트 화면에 반동이 즉시 적용되는지 확인
- [x] UE 5.8 `TeamProject_FinEditor` 빌드 확인

## 남은 작업 또는 확장 방향

* 파츠, 등급에 따른 반동/Spread 보정
* UI 조준점 크기와 현재 Spread 값 연동