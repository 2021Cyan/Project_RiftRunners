# Issue #17 발사 시스템 구현

## 구현 목적

GAS 기반 발사 Ability에서 서버 권한으로 무기 발사를 실행하고, 무기 설정에 따라 히트스캔 또는 투사체 방식으로 처리한다.

발사 시 공통으로 탄약, 발사 간격, 데미지 적용, GameplayCue 기반 피드백, 네트워크 복제를 고려한다.

## 주요 구현 내용

### 발사 흐름

1. 플레이어 입력에서 `GA_Fire`를 활성화한다.
2. `GA_Fire`는 서버에서 `UWeaponComponent::ExecuteFire`를 호출한다.
3. `UWeaponComponent`는 현재 장착된 `AWeaponBase`에 발사를 위임한다.
4. `AWeaponBase`는 `RuntimeStats.FireType`에 따라 히트스캔 또는 투사체 발사를 선택한다.
5. 발사 성공 시 탄약을 차감하고 발사 간격을 기록한다.

### 히트스캔 발사

`EWeaponFireType::HitScan` 무기는 즉시 라인 트레이스로 피격 판정을 처리한다.

- 발사 시작점: `FirePointSocket` 우선, 없으면 플레이어 액터 위치
- 조준 방향: 플레이어 카메라 기준 목표 지점으로 보정
- 데미지: 대상 ASC와 GameplayEffect가 있을 때 GAS 데미지 적용
- 피드백: 발사/피격 GameplayCue 실행
- 시각 탄도: 실제 탄환 액터 없이 Fire GameplayCue에서 `HitScanTracerEffect`를 순간 재생

### 투사체 발사

`EWeaponFireType::Projectile` 무기는 `AWeaponProjectile` 액터를 풀에서 꺼내 발사한다.

- `UProjectilePoolComponent`로 투사체 재사용
- `UProjectileMovementComponent`로 이동 처리
- 무기별 투사체 속도, 수명, 중력 적용 여부 설정
- 투사체 비행 중 `ProjectileTrailEffect`를 투사체 액터에 부착
- 충돌 시 GAS 데미지와 Impact GameplayCue 실행
- 수명 만료 또는 충돌 처리 후 풀로 반환

투사체는 실제 액터이므로 서버에서 스폰/활성화되고 movement replication을 사용한다.

### 투사체 확장 설정

`FWeaponRuntimeStats`에 투사체 확장용 설정을 포함한다.

- `ProjectileSpeed`: 투사체 이동 속도
- `ProjectileLifeSeconds`: 풀 반환 전 생존 시간
- `bProjectileAffectedByGravity`: 중력 적용 여부
- `ProjectileGravityScale`: 중력 배율
- `bProjectileShouldBounce`: 바운스 사용 여부
- `ProjectileBounciness`: 바운스 탄성
- `bProjectileExplodeOnImpact`: 충돌 즉시 데미지/반환 여부
- `ExplosionRadius`: 0보다 크면 충돌 지점 기준 반경 GAS 데미지
- `bProjectileHomingEnabled`: 호밍 사용 여부
- `ProjectileHomingAcceleration`: 호밍 가속도

현재 호밍 타겟 자동 탐색은 구현하지 않고, `AWeaponProjectile::SetHomingTarget`으로 이후 자동조준 이슈에서 연결할 수 있게 열어두었다.

### Trail / Tracer / Explosion

탄도 시각효과는 발사 방식에 따라 다르게 처리한다.

- `HitScanTracerEffect`: 히트스캔 전용. Fire GameplayCue에서 발사 시작점부터 피격점 또는 사거리 끝까지 순간 재생한다.
- `ProjectileTrailEffect`: 투사체 전용. `AWeaponProjectile`의 NiagaraComponent에 붙어서 투사체가 살아있는 동안 재생된다.
- `ExplosionEffect`: 유탄처럼 충돌 지점에서 폭발 표현이 필요한 투사체 전용. Impact GameplayCue에서 `ExplosionRadius`가 0보다 클 때 우선 재생된다.
- `ExplosionSound`: 폭발 전용 사운드. 없으면 기존 `ImpactSound`를 사용한다.

히트스캔 tracer Niagara는 끝점을 받을 수 있어야 한다. Fire GameplayCue는 생성된 NiagaraComponent에 다음 User 파라미터를 모두 넣는다.

- `User.TraceEnd`
- `User.BeamEnd`
- `User.EndLocation`

Niagara 에셋에서는 이 중 하나를 사용해 Beam/Ribbon 끝점을 연결하면 된다.

PCG 조합마다 Trail을 다르게 두지는 않는다. `RifleFeedback`, `SniperFeedback`, `GrenadeLauncherFeedback`처럼 무기 타입 단위로 전용 Trail/Tracer/Explosion을 지정한다.

### 피드백 구조

VFX/SFX는 GameplayCue Notify에서 실행한다.

- `GameplayCue.Weapon.Fire`: 발사 몽타주, muzzle flash, 탄피 배출, 발사 사운드
- `GameplayCue.Weapon.Impact.Surface`: 표면 피격 이펙트/사운드
- `GameplayCue.Weapon.Impact.Character`: 캐릭터 피격 이펙트/사운드

소유 클라이언트는 `GA_Fire`의 `LocalPredicted` 실행 중 Fire GameplayCue를 즉시 예측 재생한다. 이 Cue는 발사음과 머즐 플래시 같은 입력 피드백을 지연 없이 보여주기 위한 것이며, 실제 히트 판정, 탄약 차감, 투사체 스폰은 서버의 `AWeaponBase::ExecuteFire`에서 확정한다.

서버에서 실행되는 Fire GameplayCue는 다른 클라이언트가 해당 발사를 듣고 볼 수 있게 전파하는 역할을 한다. 클라이언트에서 무기 Actor 포인터가 아직 해석되지 않은 경우에도 `WeaponDataAsset`과 캐릭터의 `WeaponComponent`를 이용해 발사음을 위치 기반으로 재생할 수 있게 fallback을 둔다.

`UWeaponDataAsset`은 무기 타입별 피드백 세트를 3개 가진다.

- `RifleFeedback`
- `SniperFeedback`
- `GrenadeLauncherFeedback`

각 피드백 세트는 muzzle flash, shell ejection, surface impact, character impact, hitscan tracer, projectile trail, explosion effect, fire sound, impact sound, explosion sound, volume, pitch range, attenuation을 가진다.

### 무기 데이터

무기 외형은 `AssemblyData`만 사용한다.

- `FrameMesh`
- `BarrelMesh`
- `BarrelSocketName`
- `BarrelRelativeTransform`

추후 PCG 무기 생성은 조합 결과 메시를 `AssemblyData`에 채워서 `AWeaponBase`가 그대로 사용하게 한다.

## 관련 구현

- `Source/TeamProject_Fin/GameCore/Ability/GA_Fire.h`
- `Source/TeamProject_Fin/GameCore/Ability/GA_Fire.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/Core/WeaponComponent.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Core/WeaponComponent.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/Core/WeaponBase.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Core/WeaponBase.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/Projectiles/WeaponProjectile.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Projectiles/WeaponProjectile.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/Projectiles/ProjectilePoolComponent.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Projectiles/ProjectilePoolComponent.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/Libraries/WeaponFireLibrary.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Libraries/WeaponFireLibrary.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/Libraries/WeaponDamageLibrary.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Libraries/WeaponDamageLibrary.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/Libraries/WeaponCueLibrary.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Libraries/WeaponCueLibrary.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/GameplayCues/GCN_WeaponFire.h`
- `Source/TeamProject_Fin/GameCore/Weapon/GameplayCues/GCN_WeaponFire.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/GameplayCues/GCN_WeaponImpact.h`
- `Source/TeamProject_Fin/GameCore/Weapon/GameplayCues/GCN_WeaponImpact.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/Data/WeaponDataAsset.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Data/WeaponDataAsset.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/Data/WeaponTypes.h`

## 사용 방법

### 공통 세팅

1. 플레이어의 `DefaultAbilities`에 `GA_Fire`를 추가한다.
2. `AWeaponBase` 기반 Blueprint를 준비한다.
3. `UWeaponDataAsset` 기반 무기 데이터 에셋을 만든다.
4. `WeaponDataAsset.AssemblyData`에 프레임/배럴 메시를 지정한다.
5. 배럴 메시에는 `FirePointSocket`을 만든다.
6. 프레임 메시에는 탄피 배출용 `ShellEjectSocket`을 만든다.
7. GameplayCue Notify Blueprint는 `/Game/GameplayCues` 아래에 둔다.

### 히트스캔 무기

`WeaponDataAsset.RuntimeStats.FireType`을 `HitScan`으로 설정한다.

라이플 또는 저격총은 보통 이 방식을 사용한다.

### 투사체 무기

`WeaponDataAsset.RuntimeStats.FireType`을 `Projectile`로 설정한다.

필수 설정:

- `ProjectileClass`: `AWeaponProjectile` 기반 Blueprint
- `ProjectileSpeed`: 투사체 속도
- `ProjectileLifeSeconds`: 투사체 생존 시간

유탄처럼 포물선 이동이 필요하면:

- `bProjectileAffectedByGravity = true`
- `ProjectileGravityScale` 조정
- `ExplosionRadius`를 0보다 크게 설정
- `GrenadeLauncherFeedback.ExplosionEffect`와 `ExplosionSound` 지정

직선 탄환처럼 중력이 없어야 하면:

- `bProjectileAffectedByGravity = false`

추후 바운스 유탄 준비:

- `bProjectileShouldBounce = true`
- `ProjectileBounciness` 조정
- 필요하면 `bProjectileExplodeOnImpact = false`로 즉시 폭발을 막고 별도 폭발 조건을 추가한다.

히트스캔 시각 tracer 설정:

- `RifleFeedback.HitScanTracerEffect`
- `SniperFeedback.HitScanTracerEffect`

투사체 trail 설정:

- `GrenadeLauncherFeedback.ProjectileTrailEffect`

## 테스트 및 확인 내용

- [x] 에디터에서 플레이어 Blueprint의 `DefaultAbilities`에 `BP_GA_Fire`가 있는지 확인
- [x] 테스트 무기 데이터의 `RuntimeStats.FireType`을 `HitScan` 또는 `Projectile`로 바꿔 각각 확인
- [x] `WeaponComponent.bShowDebug`로 발사 시작점과 방향이 시각 디버그로 표시됨을 확인
- [x] 투사체는 발사 시작점과 짧은 방향 선이 표시되고, 실제 투사체 액터가 이동함을 확인
- [x] 탄약이 차감되고, 발사 버튼을 누르고 있는 동안 `FireInterval`에 맞춰 연사되는지 확인

## 남은 작업 또는 확장 방향

- 이슈 #18 장전 시스템과 탄약 UI 연동
- 이슈 #19 반동/스프레드 실제 적용
- 이슈 #21 자동조준과 `SetHomingTarget` 연결
- 이슈 #23 유탄 바운스 후 폭발 조건 구현
- 이슈 #30 장착/교체 흐름 정식 연결
- GameplayCue 에셋별 Niagara/Sound 실제 리소스 배치
