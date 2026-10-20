# #5 GameplayEffect 기본 구조 구현

## Issue

- Issue 번호: #5
- 기능명: GameplayEffect 기본 구조 구현
- 담당자: 2021Cyan
- 관련 Sprint: 1주차

## 목적

데미지, 회복, 스태미나 소모, 쿨다운을 GAS GameplayEffect 기반으로 적용할 수 있게 공통 구조를 만듭니다.

## 구현 내용

- ASC에서 GameplayEffect Spec을 생성하고 적용하는 공통 패턴 구성
- `DamageSystemLibrary`와 `FDamagePayload` 기반의 통합 데미지 경로 구성
- SetByCaller Data 태그로 데미지, 회복, 스태미나 수치 전달
- GameplayAbility의 Cost/Cooldown GameplayEffect와 `CommitAbility` 기반 처리 구성
- GE Blueprint Asset 생성
  - `GE_Damage_Base`
  - `GE_Heal`
  - `GE_StaminaCost`
  - `GE_Cooldown`

## 관련 구현

- C++: `DamageSystemLibrary`, `GameplayAbility`, `AbilitySystemComponent`
- Blueprint: `GE_Damage_Base`, `GE_Heal`, `GE_StaminaCost`, `GE_Cooldown`
- Asset: 

## 사용 방법

데미지는 `FDamagePayload`에 양수로 설정하고 서버에서 `DamageSystemLibrary`를 통해 적용합니다.

```cpp
FDamagePayload Payload;
Payload.BaseDamage = 25.0f;
Payload.HitResult = HitResult;
Payload.DamageEffectClass = DamageEffectClass;
Payload.AbilityLevel = AbilityLevel;

UDamageSystemLibrary::ApplyGameDamage(this, InstigatorActor, Payload);
```

회복과 패시브 이펙트는 ASC에서 Spec을 생성한 뒤 자신에게 적용합니다.

```cpp
FGameplayEffectContextHandle Context = ASC->MakeEffectContext();
FGameplayEffectSpecHandle SpecHandle = ASC->MakeOutgoingSpec(
    HealEffectClass,
    AbilityLevel,
    Context);

if (SpecHandle.IsValid())
{
    SpecHandle.Data->SetSetByCallerMagnitude(
        GameplayTagSystem::Data_Heal,
        20.0f);
    ASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
}
```

스태미나 소모와 쿨다운은 GameplayAbility에 설정된 Cost/Cooldown GameplayEffect를 `CommitAbility`로 확인하고 적용합니다.

```cpp
if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
{
    EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
    return;
}
```

쿨다운 여부 확인:

```cpp
const bool bOnCooldown =
    ASC->HasMatchingGameplayTag(GameplayTagSystem::Cooldown_Skill_Main);
```

## 테스트 및 확인 내용

- [x] DamageSystem을 통한 데미지 GE 적용 시 `Health` 감소 확인
- [x] 회복 GE 적용 시 `Health` 증가 확인
- [x] 스태미나 소모 GE 적용 시 `Stamina` 감소 확인
- [x] `Health`, `Stamina` Clamp 확인
- [x] 쿨다운 태그 적용 후 Duration 종료 시 제거 확인
- [x] 관련 Issue 완료 조건 확인

## 남은 작업

- 없음

## 확장 방향

- 스킬별 GameplayAbility에 Cost/Cooldown GameplayEffect를 설정해 비용과 쿨다운을 적용합니다.
- 추가 스킬이 생기면 스킬별 쿨다운 태그 또는 슬롯별 쿨다운 태그를 확장합니다.
- 버프/디버프 GE가 필요하면 Data 태그와 Effect 태그를 추가해 같은 구조로 확장합니다.
