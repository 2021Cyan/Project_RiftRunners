# #23 유탄 투사체 바운스 구현

## Issue

- Issue 번호: #23
- 기능명: 유탄 투사체 바운스 구현
- 담당 영역: Weapon / Projectile / GameplayCue
- 관련 의존성: #17 발사 시스템

## 목적

유탄 투사체가 지형에 닿았을 때 즉시 폭발하지 않고 자연스럽게 튕기도록 한다.

캐릭터에 직접 충돌하면 적, 아군, 자신 여부와 관계없이 즉시 폭발하고, 바닥 또는 벽 같은 지형에 충돌하면 설정된 최대 바운스 횟수까지 튕긴 뒤 폭발한다. 각 바운스 후에는 속도를 줄여 실제 유탄처럼 점점 힘이 빠지는 움직임을 만든다.

## 주요 구현 내용

### 충돌 대상별 폭발 조건

`AWeaponProjectile`의 충돌 처리를 캐릭터 충돌과 지형 충돌로 분리했다.

- 캐릭터 충돌: `bProjectileExplodeOnImpact`가 켜져 있으면 즉시 폭발
- 지형 충돌: 바운스가 켜져 있으면 즉시 폭발하지 않고 바운스 처리
- 일반 비바운스 투사체: 기존처럼 충돌 즉시 폭발 가능

캐릭터 판정은 기존 `WeaponFireLibrary::IsCharacterHit()`를 재사용한다. 유탄은 발사자를 이동 무시 대상으로 등록하지 않으므로 자신의 캐릭터와 다시 충돌해도 즉시 폭발할 수 있다.

유탄 폭발 및 직접 충돌 피해는 발사자도 피해 대상에 포함하도록 `WeaponDamageLibrary`의 Instigator 피해 허용 옵션을 사용한다. 이 옵션은 유탄 투사체 경로에서만 켜며, 기존 히트스캔과 다른 광역 피해 로직은 기존처럼 발사자를 제외한다.

### 최대 바운스 횟수

`FWeaponRuntimeStats`에 `MaxProjectileBounces`를 추가했다.

기본값은 3이며, 유탄이 지형에 세 번째로 충돌하면 해당 충돌 위치에서 폭발한다. 값은 0 이상으로 보정된다.

### 바운스 속도 감쇠

`FWeaponRuntimeStats`에 `ProjectileBounceSpeedMultiplier`를 추가했다.

바운스 후 투사체 속도에 이 배율을 곱한다. 기본값은 0.75이며, 값은 0.0~1.0 범위로 보정된다.

### 바운스 사운드

`FWeaponFeedbackSet`에 바운스 전용 사운드 설정을 추가했다.

- `BounceSounds`
- `BounceVolume`
- `BounceSoundAttenuation`

유탄은 `WeaponDataAsset.GrenadeLauncherFeedback` 안의 Bounce Sound 항목을 사용한다. 바운스 사운드는 기존 무기 Impact GameplayCue 경로로 실행되므로 서버에서 발생한 바운스가 클라이언트에도 전파된다.

`BounceSoundAttenuation`이 지정되어 있으면 바운스 사운드에만 해당 감쇠 설정을 사용한다. 비어 있으면 기존 `SoundAttenuation`을 대신 사용한다.

바운스 Cue는 사운드만 재생하고 표면 Impact Niagara는 재생하지 않는다. 마지막 바운스에서 폭발할 때는 Bounce Sound 대신 기존 Explosion Sound와 Explosion Effect를 사용한다.

## 관련 구현

- `Source/TeamProject_Fin/GameCore/Weapon/Projectiles/WeaponProjectile.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Projectiles/WeaponProjectile.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/Data/WeaponTypes.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Data/WeaponDataAsset.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Core/WeaponBase.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/Libraries/WeaponCueLibrary.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Libraries/WeaponCueLibrary.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/GameplayCues/GCN_WeaponImpact.cpp`

## 사용 방법

유탄 무기 데이터 에셋에서 다음 값을 설정한다.

1. `RuntimeStats.FireType = Projectile`
2. `RuntimeStats.bProjectileShouldBounce = true`
3. `RuntimeStats.bProjectileExplodeOnImpact = true`
4. `RuntimeStats.MaxProjectileBounces = 3`
5. `RuntimeStats.ProjectileBounceSpeedMultiplier = 0.75`
6. `RuntimeStats.ProjectileBounciness`와 `ProjectileGravityScale`을 원하는 포물선 감각에 맞게 조정
7. `GrenadeLauncherFeedback.BounceSounds`에 바운스 사운드 지정
8. `GrenadeLauncherFeedback.BounceVolume` 조정
9. `GrenadeLauncherFeedback.BounceSoundAttenuation`에 바운스용 Sound Attenuation 지정
10. `GrenadeLauncherFeedback.ExplosionEffect`와 `ExplosionSounds` 지정

`MaxProjectileBounces`를 낮추면 더 빨리 폭발하고, `ProjectileBounceSpeedMultiplier`를 낮추면 바운스 후 속도가 더 크게 감소한다.

## 테스트 및 확인 내용

- [x] 유탄이 적 캐릭터에 직접 충돌하면 즉시 폭발하는지 확인
- [x] 유탄이 자신 또는 아군 캐릭터에 충돌하면 즉시 폭발하고 Friendly Fire 처리가 적용되는지 확인
- [x] 유탄이 바닥 또는 벽에 충돌하면 즉시 폭발하지 않고 튕기는지 확인
- [x] 기본 설정에서 n 번째 지형 충돌 시 폭발하는지 확인
- [x] 바운스할 때마다 이동 속도가 감소하는지 확인
- [x] 바운스 시 `GrenadeLauncherFeedback.BounceSounds`의 사운드가 `BounceSoundAttenuation` 거리 감쇠로 재생되는지 확인
- [x] 마지막 바운스 폭발 시 Bounce Sound가 아니라 Explosion Sound가 재생되는지 확인
- [x] 리슨 서버와 클라이언트에서 바운스 사운드와 폭발 이펙트가 모두 보이는지 확인

## 남은 작업 또는 확장 방향

