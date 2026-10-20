# #21 돌격병 과부하 스킬 구현

## Issue

- Issue 번호: #21
- 기능명: 돌격병 과부하 스킬 구현
- 담당 영역: Character Skill / GAS / GameplayCue
- 관련 의존성: #17 발사 시스템, #20 돌격병 클래스

## 목적

돌격병의 고유 스킬인 과부하(Overdrive)를 Gameplay Ability System 기반으로 구현한다.

스킬이 활성화된 동안 무기의 연사 속도와 재장전 속도를 변경하고, 같은 적을 일정 횟수 타격하면 해당 적 위치에서 전기 폭발과 연쇄 번개를 발생시킨다. 스킬 발동, 무기 루프, 폭발, 연쇄 번개의 시각 및 청각 피드백은 GameplayCue로 분리한다.

또한 캐릭터별 고유 스킬이 공통 정책과 쿨다운 구조를 재사용할 수 있도록 `UGA_CharacterSkillBase`를 추가한다.

## 주요 구조

```text
UGA_CharacterSkillBase
 └─ UGA_Overdrive
     ├─ UGE_WeaponTimingModifier
     │   └─ UCharacterAttributeSet
     │       ├─ FireIntervalMultiplier
     │       └─ ReloadMontageRateMultiplier
     ├─ Event.Weapon.Hit
     │   └─ 적별 타격 횟수 누적
     ├─ 전기 폭발
     │   └─ 범위 피해
     ├─ 연쇄 번개
     │   └─ 무작위 적 탐색 및 체인 피해
     └─ GameplayCue
         ├─ Activate
         ├─ Weapon
         ├─ Explosion
         └─ ChainLightning
```

## 구현 파일

### 스킬 및 전투 로직

- `Source/TeamProject_Fin/GameCore/Ability/GA_CharacterSkillBase.h`
- `Source/TeamProject_Fin/GameCore/Ability/GA_CharacterSkillBase.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_Overdrive.h`
- `Source/TeamProject_Fin/GameCore/Ability/GA_Overdrive.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/CombatTargetLibrary.h`
- `Source/TeamProject_Fin/GameCore/Ability/CombatTargetLibrary.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/Libraries/WeaponDamageLibrary.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Libraries/WeaponDamageLibrary.cpp`

### 무기 속도 변경

- `Source/TeamProject_Fin/GameCore/Effect/GE_WeaponTimingModifier.h`
- `Source/TeamProject_Fin/GameCore/Effect/GE_WeaponTimingModifier.cpp`
- `Source/TeamProject_Fin/GameCore/Attribute/CharacterAttributeSet.h`
- `Source/TeamProject_Fin/GameCore/Attribute/CharacterAttributeSet.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/Core/WeaponBase.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_Reload.cpp`

### GameplayCue

- `Source/TeamProject_Fin/GameCore/Ability/GameplayCues/GameplayCueFxLibrary.h`
- `Source/TeamProject_Fin/GameCore/Ability/GameplayCues/GameplayCueFxLibrary.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GameplayCues/GCN_AttachNiagaraAndSound.h`
- `Source/TeamProject_Fin/GameCore/Ability/GameplayCues/GCN_AttachNiagaraAndSound.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GameplayCues/GCN_SpawnNiagaraAndSound.h`
- `Source/TeamProject_Fin/GameCore/Ability/GameplayCues/GCN_SpawnNiagaraAndSound.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GameplayCues/GCN_SplineNiagaraAndSound.h`
- `Source/TeamProject_Fin/GameCore/Ability/GameplayCues/GCN_SplineNiagaraAndSound.cpp`

### 캐릭터 등록 및 태그

- `Source/TeamProject_Fin/GameCore/Character/BaseCharacter.h`
- `Source/TeamProject_Fin/GameCore/Character/PlayerCharacter.h`
- `Source/TeamProject_Fin/GameCore/Character/PlayerCharacter.cpp`
- `Source/TeamProject_Fin/GameCore/Tags/GameplayTagSystem.h`
- `Source/TeamProject_Fin/GameCore/Tags/GameplayTagSystem.cpp`

### 에셋

- `Content/GameCore/Weapon/GA/BP_GA_Overdrive.uasset`
- `Content/GameCore/GE/GE_Cooldown.uasset`
- `Content/GameplayCues/GC_Skill_Overdrive_Activate.uasset`
- `Content/GameplayCues/GC_Skill_Overdrive_Weapon.uasset`
- `Content/GameplayCues/GC_Skill_Overdrive_Explosion.uasset`
- `Content/GameplayCues/GC_Skill_Overdrive_ChainLightning.uasset`

## 공통 캐릭터 스킬 베이스

`UGA_CharacterSkillBase`는 캐릭터별 고유 스킬이 공유하는 GAS 설정을 관리한다.

기본 설정:

- Instancing Policy: `Instanced Per Actor`
- Net Execution Policy: `Local Predicted`
- 입력 직접 복제 활성화
- Ability 태그: `Ability.Skill.Main`
- 활성 중 부여 태그: `State.Skill.Active`
- 활성 차단 태그: `State.Dead`
- 기본 쿨다운: 60초

쿨다운 시간은 `CooldownDuration`으로 조정하며, `ApplyCooldown()`에서 `Data.Cooldown` SetByCaller 값으로 공용 `GE_Cooldown`에 전달한다. 실제 쿨다운 적용은 자식 Ability가 `CommitAbility()`를 호출할 때 실행된다.

```cpp
if (!CommitAbility(Handle, ActorInfo, ActivationInfo)) {
	EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
	return;
}
```

## Overdrive 실행 흐름

```text
고유 스킬 입력
  -> BP_GA_Overdrive 활성화
  -> CommitAbility로 비용 및 쿨다운 적용
  -> State.Skill.Active + State.Skill.Overdrive 부여
  ├─ 소유 클라이언트
  │   ├─ 발동 Cue 예측 실행
  │   └─ 무기 루프 Cue 예측 추가
  └─ 서버
      ├─ 무기 속도 GameplayEffect 적용
      ├─ 발동/무기 Cue를 다른 클라이언트에 전파
      └─ Event.Weapon.Hit 수신 시작
  -> 클라이언트/서버 Duration 경과
  -> 서버에서 무기 루프 Cue 및 속도 효과 제거
  -> 서버 타격 카운터 초기화
  -> 클라이언트/서버 Ability 종료
```

`State.Skill.Overdrive`가 이미 있거나 캐릭터가 사망 상태이면 스킬을 다시 활성화할 수 없다.

Ability 자체는 `LocalPredicted`로 실행한다. 소유 클라이언트는 입력 즉시 Activation/Weapon Cue를 재생하고, 서버가 같은 Prediction Key로 전파한 Cue는 GAS가 중복 실행되지 않도록 처리한다. 무기 배율, 타격 횟수, 폭발 대상, 체인 대상과 피해는 서버가 확정한다.

## 무기 속도 변경

과부하의 배율은 서버가 `UGE_WeaponTimingModifier`를 통해 Attribute에 적용한다.

| 항목 | 기본값 | 의미 |
|---|---:|---|
| `FireIntervalMultiplier` | `0.5` | 발사 간격을 절반으로 줄여 연사 속도 증가 |
| `ReloadMontageRateMultiplier` | `1.5` | 재장전 몽타주 재생 속도 증가 |

배율은 하드코딩하지 않고 `BP_GA_Overdrive`의 Class Defaults에서 조정할 수 있다.

`AWeaponBase`는 발사 시 이미 계산된 `FireIntervalMultiplier` Attribute를 사용하고, `UGA_Reload`는 재장전 시작 시 `ReloadMontageRateMultiplier` Attribute를 사용한다. 이 값은 GameplayEffect가 적용되거나 제거될 때 Attribute 값이 변경된다. 연사 및 재장전 배율 GameplayEffect는 서버에서 적용하며, 변경된 배율 Attribute는 클라이언트에 복제된다. 발동 및 무기 이펙트는 입력 즉시 보이도록 클라이언트에서 예측 재생한다.

스킬 종료 또는 취소 시 저장해 둔 `FActiveGameplayEffectHandle`로 속도 효과를 제거하므로 원래 무기 속도로 복귀한다.

## 적별 타격 누적과 전기 폭발

일반 무기 피해가 성공하면 `WeaponDamageLibrary`가 공격자의 ASC로 범용 이벤트 `Event.Weapon.Hit`을 전달한다. Overdrive는 활성 중에만 이 이벤트를 수신한다.

타격 횟수는 다음 구조로 적마다 따로 저장한다.

```cpp
TMap<TWeakObjectPtr<AActor>, int32> HitCountsByTarget;
```

기본 `HitsToExplosion`은 10이다. 적 A를 8회 타격한 뒤 적 B를 2회 타격해도 폭발하지 않으며, 같은 적이 10회 누적되었을 때만 해당 적의 피격 위치에서 폭발한다. 폭발한 적의 카운터만 0으로 초기화된다.

폭발은 `ExplosionRadius` 안의 적대 대상을 탐색하고 `ExplosionDamageEffectClass`로 GAS 피해를 적용한다. 기본값은 반경 500, 피해량 100이다. 폭발 Cue는 실제 타격 위치를 전달받아 그 위치에서 실행된다.

## 연쇄 번개

전기 폭발 후 `CombatTargetLibrary::SelectRandomHostileTargets()`로 주변 적을 무작위 탐색한다.

기본 설정:

| 항목 | 기본값 | 의미 |
|---|---:|---|
| `ChainSearchRadius` | `1200` | 전체 체인 후보 탐색 범위 |
| `ChainMaxDistance` | `600` | 현재 대상에서 다음 대상까지 허용 거리 |
| `MaxChainTargets` | `4` | 최대 추가 연결 대상 수 |
| `ChainDamage` | `50` | 연결된 대상에게 적용할 피해 |
| `bRequireChainLineOfSight` | `false` | 대상 사이 시야 확보 요구 여부 |

이미 연결된 대상은 다시 선택하지 않으며, 아군과 사망한 대상은 제외한다. `bRequireChainLineOfSight`가 켜져 있으면 `ECC_Visibility` 라인 트레이스로 가려진 대상을 제외한다.

각 체인은 이전 대상 Actor와 다음 대상 Actor를 Cue 파라미터로 전달한다. `AGCN_SplineNiagaraAndSound`는 두 Actor의 위치를 추적하므로 대상이 움직여도 Spline 끝점이 다음 대상 위치를 따라간다. 중간 제어점에는 수평 및 수직 무작위 오프셋을 주어 번개 형태를 만든다.

체인이 여러 번 연결되어도 사운드는 첫 번째 링크에서만 재생한다. Cue 에셋에서 여러 사운드 중 하나와 최소/최대 Pitch를 무작위로 선택할 수 있다.

각 Chain Cue는 연결된 대상의 ASC에서 실행한다. 모든 링크를 공격자 ASC 하나에서 같은 프레임에 실행할 때 발생하는 `net.MaxRPCPerNetUpdate` 제한을 피하고, 각 링크를 원격 클라이언트에 빠짐없이 전파하기 위한 구조다.

## GameplayCue 구성

공용 Cue 부모는 특정 스킬에 종속되지 않으며 Niagara와 Sound 설정만 에셋에서 교체해 재사용한다.

| Cue 태그 | 실행 기준 | 권장 부모 | 위치 | 수명 및 사운드 |
|---|---|---|---|---|
| `GameplayCue.Skill.Overdrive.Activate` | 클라이언트 예측 + 서버 전파 | `GCN_AttachNiagaraAndSound` | `TargetFeet` | 발동 이펙트가 끝날 때까지 캐릭터 발 위치 추적 |
| `GameplayCue.Skill.Overdrive.Weapon` | 클라이언트 예측 + 서버 전파 | `GCN_AttachNiagaraAndSound` | `EquippedWeapon` | 총구 위치에 루프, 스킬 종료 시 즉시 제거 |
| `GameplayCue.Skill.Overdrive.Explosion` | 서버 확정 | `GCN_SpawnNiagaraAndSound` | 전달된 타격 위치 | 일회성 재생, 폭발음 목록에서 무작위 선택 가능 |
| `GameplayCue.Skill.Overdrive.ChainLightning` | 서버 확정, 대상 ASC별 실행 | `GCN_SplineNiagaraAndSound` | 시작/종료 Actor 사이 | Spline Niagara, 첫 링크 사운드만 재생 |

`GameplayCueFxLibrary`는 발 위치 계산, Niagara 생성, 랜덤 Sound와 Pitch 선택 등 Cue가 공통으로 사용하는 처리를 제공한다.

### 디버그 표시

`BP_GA_Overdrive`에서 `bShowDebug`를 활성화하면 Cue 파라미터에 `CueParameter.ShowDebug`가 포함된다.

- 전기 폭발 범위: 디버그 구체
- 연쇄 번개 경로: 디버그 Spline

`CueParameter.PlaySound`는 체인 Cue에서 첫 번째 링크만 소리를 재생하도록 전달하는 제어 태그다.

## GameplayTag

태그는 ini에 직접 추가하지 않고 프로젝트 규칙에 따라 `GameplayTagSystem.h/.cpp`에 Native GameplayTag로 선언한다.

| 태그 | 용도 |
|---|---|
| `Ability.Skill.Main` | 캐릭터 고유 스킬 공통 분류 |
| `State.Skill.Active` | 캐릭터 고유 스킬 활성 상태 |
| `State.Skill.Overdrive` | 과부하 스킬 활성 상태 |
| `Cooldown.Skill.Main` | 캐릭터 고유 스킬 공통 쿨다운 |
| `Data.Cooldown` | 쿨다운 시간을 GE에 전달하는 SetByCaller 키 |
| `Data.Multiplier.FireInterval` | 발사 간격 배율 전달 |
| `Data.Multiplier.ReloadMontageRate` | 재장전 속도 배율 전달 |
| `Event.Weapon.Hit` | 무기 타격 범용 GameplayEvent |

## 에디터 설정 방법

### 1. 공용 `GE_Cooldown` 설정

`GE_Cooldown`은 모든 캐릭터 고유 스킬이 공유한다.

1. Duration Policy를 `Has Duration`으로 설정한다.
2. Duration Magnitude를 `Set By Caller`로 설정한다.
3. Data Tag에 `Data.Cooldown`을 지정한다.
4. `Grant Tags to Target Actor` 컴포넌트를 추가한다.
5. Granted Tags에 `Cooldown.Skill.Main`을 추가한다.

`Cooldown Gameplay Effect Class grants no tags` 경고가 나오면 4~5번 설정이 빠진 것이다.

### 2. `BP_GA_Overdrive` 설정

1. 부모 클래스를 `GA_Overdrive`로 지정한다.
2. `Cooldown Gameplay Effect Class`에 `GE_Cooldown`을 지정한다.
3. `Cooldown Duration`을 기본 60초 또는 원하는 값으로 설정한다.
4. `Explosion Damage Effect Class`에 일반 GAS Damage GameplayEffect를 지정한다.
5. `Chain Damage Effect Class`에 체인 피해용 GameplayEffect를 지정한다.
6. Overdrive, Weapon, Electric Explosion, Chain Lightning 항목의 수치를 조정한다.

피해 Effect Class는 피해량을 전달하고 대상의 ASC에 실제 피해를 적용하기 위해 필요하다. Niagara와 Sound를 재생하는 GameplayCue는 시각 및 청각 표현만 담당한다.

### 3. GameplayCue 에셋 설정

각 에셋에서 `Gameplay Cue Tag`를 표의 태그와 일치시킨다.

#### `GC_Skill_Overdrive_Activate`

- 부모: `GCN_AttachNiagaraAndSound`
- Attach Target: `TargetFeet`
- Niagara: 발동 Niagara System
- Sound: 발동 Sound

#### `GC_Skill_Overdrive_Weapon`

- 부모: `GCN_AttachNiagaraAndSound`
- Attach Target: `EquippedWeapon`
- Niagara: 무기 루프 Niagara System
- Sound: Looping Sound

무기 Cue는 무기 발사 이펙트가 사용하는 총구 Transform과 같은 위치를 사용한다. Sound Wave 또는 Sound Cue 자체도 Loop 설정되어 있어야 한다.

#### `GC_Skill_Overdrive_Explosion`

- 부모: `GCN_SpawnNiagaraAndSound`
- Niagara: 폭발 Niagara System
- Random Sounds: 사용할 폭발음 목록
- Sound Pitch: 재생 Pitch

#### `GC_Skill_Overdrive_ChainLightning`

- 부모: `GCN_SplineNiagaraAndSound`
- Niagara: 체인 번개 Niagara System
- Spline User Parameter: `User.Spline`
- Intermediate Point Count 및 수평/수직 Jitter 조정
- Random Sounds와 Pitch Min/Max 설정

Spline을 더 부드럽게 만들려면 `Intermediate Point Count`를 늘리고 Jitter 값을 조정한다.

### 4. 돌격병 캐릭터에 등록

돌격병 PlayerCharacter Blueprint에서 다음 항목을 지정한다.

1. `DefaultAbilities`에 `BP_GA_Overdrive`를 추가한다.
2. `UniqueSkillAction`에 고유 스킬 Input Action을 지정한다.

`DefaultAbilities`는 ASC에 Ability를 Grant하는 목록이다. 고유 스킬 입력 시에는 `Ability.Skill.Main` 태그를 가진 Ability를 찾아 활성화하므로 별도의 스킬 클래스 참조는 필요하지 않다. 각 캐릭터의 `DefaultAbilities`에는 해당 캐릭터의 고유 스킬 하나만 등록한다.

## 공용 SkillBase로 새 스킬 만들기

각 캐릭터는 고유 스킬 하나를 가지며, 새 C++ 스킬은 `UGA_CharacterSkillBase`를 직접 상속한다.

```cpp
#include "GA_CharacterSkillBase.h"
#include "GA_NewCharacterSkill.generated.h"

UCLASS()
class TEAMPROJECT_FIN_API UGA_NewCharacterSkill : public UGA_CharacterSkillBase
{
	GENERATED_BODY()

protected:
	virtual void ActivateAbility(
		FGameplayAbilitySpecHandle Handle,
		const FGameplayAbilityActorInfo* ActorInfo,
		FGameplayAbilityActivationInfo ActivationInfo,
		const FGameplayEventData* TriggerEventData) override;
};
```

구현 순서:

1. `UGA_CharacterSkillBase`를 상속하는 C++ Ability를 만든다.
2. 생성자에서 해당 스킬 전용 상태 태그와 차단 태그만 추가한다.
3. `ActivateAbility()` 시작 시 `CommitAbility()`를 호출한다.
4. 종료 및 취소 경로에서 적용한 GameplayEffect, GameplayCue, AbilityTask를 정리한다.
5. C++ Ability를 부모로 하는 Blueprint Ability를 만든다.
6. Blueprint의 `Cooldown Gameplay Effect Class`에 공용 `GE_Cooldown`을 지정한다.
7. 캐릭터 Blueprint의 `DefaultAbilities`에 새 Ability를 등록한다.

C++ 클래스는 Blueprint Generated Class를 부모로 선언할 수 없으므로, 공통 로직은 C++ `UGA_CharacterSkillBase`에 두고 각 스킬 Blueprint는 자신의 C++ 스킬 클래스를 상속한다.

## 확인 항목

- [x] 프로젝트 Editor Development 빌드 성공
- [x] 스킬 사용 시 `State.Skill.Active`, `State.Skill.Overdrive` 적용 확인
- [x] 스킬 활성 중 발사 간격 및 재장전 속도 변경 확인
- [x] 종료 및 취소 시 무기 속도와 무기 루프 Cue 즉시 복구 확인
- [x] 같은 적 10회 타격 시 해당 적 위치에서만 폭발하는지 확인
- [x] 폭발 범위 피해와 최대 체인 수 및 거리 제한 확인
- [x] 체인 이동 중 Spline 끝점이 대상 Actor를 추적하는지 확인
- [x] 60초 동안 재사용이 차단되고 종료 후 다시 활성화되는지 확인
- [x] 클라이언트 스킬 사용 시 Activation/Weapon Cue 예측 재생 확인
- [x] 리슨 서버에서 발생한 모든 Chain Cue가 클라이언트에 표시되는지 확인
- [x] 서버 및 클라이언트에서 피해가 중복되지 않는지 확인

## 확장 방향

- HUD에 스킬 활성 상태와 남은 쿨다운 표시
- 캐릭터별 스킬 데이터 에셋 분리
