# #16 공통 무기 베이스 구현

## Issue

- Issue 번호: #16
- 기능명: 공통 무기 베이스 구현
- 담당 영역: Weapon / GAS 연동 기반
- 관련 의존성: #6 BaseCharacter

## 목적

모든 캐릭터가 공통 방식으로 무기를 장착하고, 이후 Gameplay Ability System(GAS)을 통해 발사와 장전을 실행할 수 있는 무기 기반 구조를 구현한다.

이번 단계에서는 실제 투사체 생성, HitScan 판정, VFX/SFX, HUD 표시는 구현하지 않는다. 대신 다음 기능들이 가능한 기반을 만든다.

현재 실제 발사 판정과 GameplayCue 피드백은 #17 발사 시스템 구현에서 연결되어 있다.

- 캐릭터가 무기 컴포넌트를 가진다.
- 무기 액터를 스폰하고 캐릭터 메시에 장착할 수 있다.
- 무기 기본 데이터는 `UWeaponDataAsset`으로 분리한다.
- 무기는 탄약, 장전 가능 여부, 발사 가능 여부를 관리한다.
- GAS Ability는 `WeaponComponent`를 통해 발사/장전 실행 함수를 호출할 수 있다.

## 주요 구조

```text
ABaseCharacter
 └─ UWeaponComponent
     └─ AWeaponBase
         ├─ UStaticMeshComponent WeaponMesh
         ├─ UStaticMeshComponent BarrelMesh
         ├─ UProjectilePoolComponent ProjectilePool
         ├─ UWeaponDataAsset WeaponData
         └─ FWeaponRuntimeStats RuntimeStats
```

## 구현 파일

- `Source/TeamProject_Fin/GameCore/Character/BaseCharacter.h`
- `Source/TeamProject_Fin/GameCore/Character/BaseCharacter.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/Data/WeaponTypes.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Data/WeaponDataAsset.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Core/WeaponBase.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Core/WeaponBase.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/Core/WeaponComponent.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Core/WeaponComponent.cpp`

## 클래스 역할

### `ABaseCharacter`

모든 캐릭터가 무기를 장착할 수 있도록 `UWeaponComponent`를 기본 서브오브젝트로 가진다.

```cpp
WeaponComponent = CreateDefaultSubobject<UWeaponComponent>(TEXT("WeaponComponent"));
```

캐릭터는 무기 세부 로직을 직접 처리하지 않고, 필요할 때 `GetWeaponComponent()`로 무기 컴포넌트에 접근한다.

### `UWeaponComponent`

캐릭터의 무기 장착 상태를 관리하는 컴포넌트다.

주요 책임:

- 기본 무기 자동 스폰
- 무기 장착 가능 여부 검사
- 무기 액터 장착
- 기존 무기 해제 및 정리
- 현재 장착 무기 접근
- GAS Ability가 호출할 발사/장전 진입점 제공
- 장착 무기 참조 Replication

주요 함수:

```cpp
AWeaponBase* SpawnAndEquipDefaultWeapon();
AWeaponBase* SpawnAndEquipWeapon(TSubclassOf<AWeaponBase> WeaponClass, const UWeaponDataAsset* WeaponData);
bool CanEquipWeapon(const AWeaponBase* Weapon) const;
bool EquipWeapon(AWeaponBase* NewWeapon);
void UnequipWeapon(bool bDestroyWeapon = false);
bool CanFire() const;
bool CanReload() const;
bool ExecuteFire();
bool ApplyReload();
float GetReloadTime() const;
```

`AllowedWeaponCategories`가 비어 있으면 모든 무기를 장착할 수 있고, 값이 있으면 해당 카테고리의 무기만 장착 가능하다.

기본 무기 자동 스폰은 서버 권한에서만 실행한다. 멀티플레이 환경에서 클라이언트가 별도 무기 액터를 생성하지 않도록 `GetOwner()->HasAuthority()` 조건을 사용한다.

`EquippedWeapon`은 Replication 대상이다. 후속 네트워크 이슈에서 `ReplicatedUsing` 기반 UI 갱신이나 장착 연출 동기화를 추가할 수 있다.

### `AWeaponBase`

월드에 스폰되는 실제 무기 액터다.

주요 책임:

- 무기 메쉬 관리
- 발사지점 관리
- 무기 데이터 적용
- 런타임 스탯 보관
- 탄창 내 현재 탄약 관리 (예비 탄약 무제한)
- 발사 가능 여부와 장전 가능 여부 판단
- 실제 발사 실행 함수 제공
- 탄창 내 현재 탄약 Replication

무기 메쉬는 프레임용 `WeaponMesh`와 배럴용 `BarrelMesh`를 사용한다. `WeaponMesh`를 RootComponent로 사용하고, `BarrelMesh`는 `AssemblyData.BarrelSocketName`에 부착한다.

```cpp
WeaponMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("WeaponMesh"));
SetRootComponent(WeaponMesh);

BarrelMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("BarrelMesh"));
BarrelMesh->SetupAttachment(WeaponMesh);
```

발사 시작점은 배럴 메시의 `FirePointSocket`을 우선 사용하고, 없으면 무기 또는 발사자 액터 위치를 사용한다.

발사 방식은 `HandleFire()`에서 확장한다. 현재 #17 구현에서는 `RuntimeStats.FireType`에 따라 HitScan 또는 Projectile 경로를 선택한다.

```cpp
virtual bool HandleFire(AActor* InstigatorActor, const FTransform& FireTransform, UAbilitySystemComponent* SourceASC, TSubclassOf<UGameplayEffect> DamageEffectClass, float AbilityLevel, bool bShowDebug);
```

`CurrentAmmo`는 Replication 대상이다. 서버의 탄약 변경 및 클라이언트의 `OnRep_Ammo`에서 `OnAmmoChanged(CurrentAmmo)`를 전달해 HUD를 갱신한다. 예비 탄약은 무제한이므로 별도 수량을 저장하거나 복제하지 않는다.

### `UWeaponDataAsset`

무기 기본 데이터를 에디터 자산으로 분리한다.

포함 데이터:

- 무기 카테고리
- 무기 조립 데이터
- 런타임 스탯
- ProjectileClass
- 무기 타입별 피드백 세트

초기에는 완성된 기본 무기 데이터를 넣는 용도로 사용한다. 이후 PCG 무기 제작이 추가되면 파츠 조합 결과 메시를 `AssemblyData`에 채우고, 계산된 스탯을 `RuntimeStats`에 주입하는 방식으로 확장한다.

### `FWeaponRuntimeStats`

무기 실행에 필요한 최종 스탯 구조체다.

포함 값:

- `Damage`
- `FireInterval`
- `ReloadTime`
- `MagazineSize`
- `Range`
- `Recoil`
- `Spread`
- `FireType`

현재 기본 무기에서는 `UWeaponDataAsset`에서 가져온 값을 사용한다. 이후 PCG 구조에서는 프레임, 탄창, 배럴, 제조사 보너스를 합산한 최종 결과를 이 구조체에 넣는다.

새 무기는 `MagazineSize`만큼 탄창을 채운다. 예비 탄약 수량은 없으며, 재장전 완료 시 현재 탄창을 `MagazineSize`까지 채운다. 교체 후 다시 주운 무기는 기존 탄창의 남은 탄약을 유지한다.

## GAS 연동 방향

무기 액터는 GAS를 직접 알지 않는다.

발사와 장전 흐름은 GameplayAbility가 담당하고, 무기는 Ability가 호출하는 실행 대상 역할만 한다.

예상 흐름:

```text
GA_WeaponFire
 ├─ ASC 상태 태그 확인
 ├─ WeaponComponent->CanFire()
 └─ WeaponComponent->ExecuteFire()
```

```text
GA_WeaponReload
 ├─ ASC 상태 태그 확인
 ├─ WeaponComponent->CanReload()
 ├─ ReloadTime 기준 WaitDelay 또는 Montage 처리
 └─ WeaponComponent->ApplyReload()
```

장전 중 상태는 `AWeaponBase` 내부 bool 값이 아니라 GAS 태그로 관리한다.

사용 예정 태그:

- `Ability.Weapon.Fire`
- `Ability.Weapon.Reload`
- `State.Weapon.Reloading`
- `Event.Weapon.Fired`
- `Event.Weapon.Reloaded`

## 테스트 방법

### 에디터 설정

1. `UWeaponDataAsset` 기반 테스트 데이터 자산을 만든다.
2. `AWeaponBase` 기반 무기 Blueprint를 만든다.
3. 캐릭터 Blueprint의 `WeaponComponent`에 값을 설정한다.

예시:

```text
DefaultWeaponClass = BP_WeaponBase
DefaultWeaponData = DA_TestRifle
bAutoEquipDefaultWeapon = true
AllowedWeaponCategories = Rifle
AttachSocketName = WeaponSocket
```

### PIE 동작 확인

테스트용 무기 데이터와 무기 Blueprint를 캐릭터의 `WeaponComponent`에 연결한 뒤 PIE에서 기본 장착, 발사, 장전 흐름을 확인했다.

확인 결과:

- 기본 무기가 정상 스폰되고 캐릭터에 장착됨
- `WeaponDataAsset`의 메쉬와 런타임 스탯이 무기에 적용됨
- 초기 탄약이 `MagazineSize` 기준으로 설정됨
- `ExecuteFire()` 호출 시 탄약이 감소함
- `FireInterval` 안에서 연속 발사 요청 시 발사가 차단됨
- `ApplyReload()` 호출 시 예비 탄약 제한 없이 탄창이 채워짐
- 탄창이 가득 찬 상태에서는 `CanReload()`가 false를 반환함
- 기본 무기 자동 스폰은 서버 권한에서만 실행되도록 제한됨

## 빌드 검증

Unreal Engine 5.8 Development Editor 빌드 성공을 확인했다.

```text
Result: Succeeded
```

## 완료 조건 확인

- [x] 무기 베이스 클래스 생성
- [x] 공통 발사 함수 구조 작성
- [x] 장전/탄약 변수 추가
- [x] 무기별 확장 가능한 `HandleFire()` 구조 제공
- [x] 캐릭터가 `WeaponComponent`를 통해 무기 장착 가능
- [x] 기본 무기 데이터 연결 구조 제공
- [x] GAS Ability에서 호출 가능한 발사/장전 진입점 제공
- [x] 기본 무기 자동 스폰 서버 권한 처리
- [x] 장착 무기 및 탄약 값 Replication 기본 처리
- [x] 무기 해제 함수 제공
- [x] 예비 탄약 수량 제거 및 무제한 재장전 적용
- [x] 빌드 성공 확인
- [x] PIE 기반 런타임 동작 확인

## 남은 작업

다음 기능은 후속 이슈에서 구현한다.

- GAS `GA_WeaponReload` 구현
- HUD 탄약 표시
- 탄약 변경 `OnRep` 이벤트 및 HUD 갱신 연결
- 무기 장착/해제 연출의 네트워크 동기화 보강

## 확장 방향

PCG 무기 제작은 추후 다음 구조로 확장한다.

```text
Frame / Magazine / Barrel / Manufacturer
 └─ Stat Modifier 합산
     └─ FWeaponRuntimeStats 생성
         └─ AWeaponBase::InitializeFromRuntimeStats()
```

현재 구조는 `AWeaponBase`가 최종 스탯인 `FWeaponRuntimeStats`를 보고 동작하도록 설계되어 있어, DataAsset 기반 기본 무기와 PCG 결과 기반 무기를 같은 실행 경로로 처리할 수 있다.
