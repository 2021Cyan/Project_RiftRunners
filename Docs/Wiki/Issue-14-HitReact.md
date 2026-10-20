#14 피격 반응 및 그로기 시스템 구현

### Issue
- **Issue 번호**: #14
- **기능명**: 피격 반응 및 그로기 시스템 통합 구현 (C++ Core & Blueprint/Animation)
- **담당자**: seok0317
- **담당 영역**: Gameplay Ability System (GAS) / Damage Pipeline / Animation Montage
- **관련 의존성**: #6 BaseCharacter, #13 공통 데미지 파이프라인(DamageSystemLibrary)

### 목적
캐릭터가 피해를 입었을 때 데미지의 성격(약한 그로기, 강한 넉백 그로기)에 따라 4방향 피격 몽타주를 분기하여 재생하고, 상황에 맞는 물리적 넉백 효과를 동기화하여 처리하는 시스템을 구축한다.

C++ 코어에서 계산된 타격 방향과 데미지 타입 데이터를 바탕으로, 블루프린트 단에서 지정된 애니메이션 몽타주와 넉백 물리량을 실행한다. 또한, 피격 시 진행 중이던 모든 행동(구르기, 조준 등)을 GAS 태그를 통해 강제 중단시키고 자연스러운 피격 모션이 출력되도록 구현한다.

### 주요 구조
- **UAbilitySystemComponent (TargetASC)**
  - `FDamagePayload` (데미지 설정: `EDamageGroggyType` 적용)
  - **`UCharacterAttributeSet`**
    - `PostGameplayEffectExecute` (그로기 태그 감지 및 이벤트 송출)
  - **`UGA_HitReact`** (피격 전용 C++ Ability)
    - `ComputeHitDirectionSection` (피격 방향 연산)
    - `UAbilityTask_ApplyRootMotionConstantForce` (강한 그로기 넉백 물리 연산)
    - `UAbilityTask_PlayMontageAndWait` (섹션 기반 몽타주 재생)
  - **`BP_GA_HitReact`** (블루프린트 어빌리티)
    - `AM_HitReact` (약한 그로기 4방향 몽타주)
    - `AM_HitReactHeavy` (강한 그로기 4방향 몽타주)
  - **GAS Tags** (Cancel / Block / Owned 설정)

### 구현 파일
- `Source/TeamProject_Fin/GameCore/Tags/GameplayTagSystem.h`
- `Source/TeamProject_Fin/GameCore/Tags/GameplayTagSystem.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/DamageSystemLibrary.h`
- `Source/TeamProject_Fin/GameCore/Ability/DamageSystemLibrary.cpp`
- `Source/TeamProject_Fin/GameCore/Attribute/CharacterAttributeSet.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_HitReact.h`
- `Source/TeamProject_Fin/GameCore/Ability/GA_HitReact.cpp`
- `(Blueprint)` BP_GA_HitReact
- `(Animation)` AM_HitReact, AM_HitReactHeavy

### 주요 구현 내용 및 시스템 역할

**C++ 코어 클래스 역할**

**`GameplayTagSystem`**
- 피격 및 그로기와 관련된 동적 태그와 이벤트 트리거 태그를 관리한다.
- `Event.HitReact` (약한 그로기 이벤트) / `Event.HitReact.Heavy` (강한 그로기 이벤트)
- `Damage.Type.Groggy` (약한 데미지 태그) / `Damage.Type.HeavyGroggy` (강한 넉백 데미지 태그)

**`DamageSystemLibrary`**
- 기획자가 지정한 데미지 페이로드(`EDamageGroggyType`)를 판별하여 동적 데미지 태그를 GameplayEffect Spec에 삽입한다.
- Enum 구조(`EDamageGroggyType`)를 도입하여, `None` / `LightGroggy` / `HeavyGroggy` 중 하나만 선택되도록 강제하여 휴먼 에러를 방지한다.

**`CharacterAttributeSet`**
- 데미지가 적용되는 순간(`PostGameplayEffectExecute`) 동적 데미지 태그를 감지하고, 피격자 스스로에게 그로기 활성화 이벤트를 송출한다.
- 블루프린트 단의 트리거 설정 누락을 대비하여, `HandleGameplayEvent` 실패 시 `TriggerAbilityFromGameplayEvent`를 직접 호출하는 2중 안전장치를 내장했다.

**`UGA_HitReact`**
- 실질적인 피격 연산과 애니메이션 명령, 넉백을 전담한다.
- C++ 생성자에서 AbilityTriggers(`Event.HitReact`) 기본 등록.
- 피격 방향(공격자 vs 피해자)의 내적을 계산하여 4방향 몽타주 섹션(Front, Back, Left, Right) 산출.
- 강한 그로기 판정 시, 공격자의 정반대 방향을 계산해 `UAbilityTask_ApplyRootMotionConstantForce` 태스크로 물리적 넉백 적용.

**블루프린트 및 애니메이션 세팅 (BP_GA_HitReact)**

**방향별 피격 애니메이션 몽타주 구성**
- **일반 피격 (`AM_HitReact`)**: 약한 그로기 발생 시 재생. 미디엄(Medium) 피격 애니메이션을 활용하여 몸이 살짝 밀리는 연출.
- **강한 피격 (`AM_HitReactHeavy`)**: 강한 넉백 시 재생. 크고 격렬한 애니메이션을 활용하여 캐릭터가 크게 휘청이는 연출.
- **섹션 분할**: 두 몽타주 모두 타임라인에 Front, Back, Left, Right 섹션을 동일한 이름으로 배치하여 C++ 연산값과 1:1 매칭되도록 설계.

**피격 어빌리티 파라미터 세팅**
- **Montage 맵핑**: Hit React Montage와 Heavy Hit React Montage 변수에 에셋 할당.
- **넉백 물리량 제어**: Knockback Force를 `200.0`, Knockback Duration을 `1.0`으로 설정하여 C++의 루트 모션 태스크가 이 값을 읽어들여 1초 동안 `200.0`의 힘으로 밀려나게 세팅.
- **Sections**: C++ 코드와 몽타주를 연결하는 식별자 텍스트 명시적 기입.

**기존 액션 강제 취소 및 상태 제어 태그 설정**
- **어빌리티 취소 (Cancel) & 차단 (Block)**: 피격되는 순간 `Ability.Movement.Dodge(구르기)`, `Sprint(달리기)`, `Aim(조준)`, `Fire(발사)`, `Reload(장전)` 등의 어빌리티가 즉시 종료되도록 설정하고 피격 종료 전까지 재입력을 차단.
- **활성화 소유 태그 (Activation Owned Tag)**: `State.Groggy` 태그를 할당하여 피격 중 캐릭터가 '그로기 상태'에 놓이게 시스템 상태를 정의.

### 관련 구현 (GAS 연동 방향)
피해 연산 및 그로기 트리거는 서버(`CharacterAttributeSet`)에서 결정된다. 피격 어빌리티의 `NetExecutionPolicy`를 `ServerInitiated`로 강제하여, 서버에서 발동된 넉백 및 몽타주 재생 명령이 클라이언트의 예측 오류 없이 강제 동기화되도록 설계했다.

**예상 흐름:**
1. `DamageSystemLibrary::ApplyGameDamage` (Enum 값 판별 후 동적 태그 주입)
2. `CharacterAttributeSet::PostGameplayEffectExecute` (태그 감지 및 이벤트 송출)
3. `UGA_HitReact::ActivateAbility` (태그 캔슬 발동 -> 4방향 연산 -> 넉백 계산 -> 몽타주 재생)

### 사용 방법 (테스트 방법)
- PIE 환경에서 `DamageSystemLibrary::ApplyGameDamage`를 호출할 때 `EDamageGroggyType`을 `LightGroggy` 또는 `HeavyGroggy`로 설정하여 데미지를 가한다.
- 캐릭터의 `BP_GA_HitReact`에 몽타주(`AM_HitReact`, `AM_HitReactHeavy`)가 정상적으로 할당되어 있는지 확인한다.

### 테스트 또는 확인 내용

- **방향별 몽타주 재생 테스트**
  - PIE 환경에서 캐릭터의 전/후/좌/우에서 Light Groggy 데미지를 가했을 때, C++ 코어가 방향을 인식하여 `AM_HitReact`의 알맞은 섹션을 재생하는지 확인 완료.
- **넉백 물리력 동기화 검증**
  - Heavy Groggy 데미지를 가했을 때, `AM_HitReactHeavy` 모션이 재생됨과 동시에 공격자의 반대 방향으로 `200.0`의 힘만큼 부드럽게 밀려나는 루트 모션 태스크 동작 확인 완료.
- **인터럽트(취소) 검증**
  - 달리기, 조준, 구르기, 무기 장전 중에 피격을 당했을 때 진행 중이던 애니메이션과 어빌리티가 즉각 중단되고 피격 모션으로 덮어씌워지는 태그 캔슬 로직 확인 완료.
- **빌드 검증**
  - Unreal Engine 5.x Development Editor C++ 빌드 성공 확인.
  - **Result:** Succeeded

### 완료 조건 확인
- [x] 피격 연산 및 그로기 트리거를 처리하는 C++ 로직 작성
- [x] 타격 방향에 따른 4방향 몽타주 섹션 산출 로직 구현
- [x] 강한 그로기 판정 시 물리적 넉백 태스크 적용
- [x] 블루프린트(`BP_GA_HitReact`)에서 재생할 몽타주 할당 및 넉백 물리량 셋업
- [x] 구르기, 조준 등 기존 액션 강제 취소를 위한 태그 캔슬 로직 설정
- [x] PIE 런타임 동작 확인 및 동기화 검증
- [x] 빌드 성공 확인

### 남은 작업 또는 확장 방향

- **피격 이펙트 및 사운드 연결**
  - 몽타주 내부의 노티파이(Anim Notify) 트랙에 피격 순간 피가 튀는 파티클(VFX)이나 타격음(SFX)을 발생시키는 `Play Particle Effect` 및 `Play Sound` 노티파이를 추가하여 타격감을 보강할 예정.
- **시스템 확장**
  - 현재 이벤트 페이로드 기반 로직이므로, 추후 C++ 단에서 `EventMagnitude`와 `HitResult`를 파싱하여 정확한 피격 부위 뼈(Bone) 위치에 다이나믹 블러드 데칼이나 이펙트를 생성하는 방향으로 확장 가능.
