# Issue #10 스태미나 시스템 코어 구현 (C++ / Blueprint 통합 편)

## Issue
- **Issue 번호**: #10
- **기능명**: 스태미나 시스템 코어 구현
- **담당자: seok0317
- **담당 영역**: GAS AttributeSet / Gameplay Tag / Ability (C++ & Blueprint)
- **관련 의존성**: #3 AttributeSet 기본 구조, #6 3인칭 캐릭터 이동 구현

## 구현 목적
다크소울, 램넌트2 등 하드코어 액션 게임의 핵심 리소스인 '스태미나' 관리 시스템을 구현한다. 구르기, 달리기 등 모든 전투 액션의 제약 조건이 되며, 특히 "스태미나 소모 후 일정 시간 동안 회복되지 않는 지연 시간(Regen Pause)"과 "스태미나 고갈 시 액션 강제 종료" 메커니즘을 틱(Tick) 연산이나 하드코딩된 조건문 없이 완벽하게 최적화하여 구현한다. C++ 코어의 안정성과 블루프린트의 유연한 GAS 데이터 세팅을 결합하여 이벤트를 기반으로 동작하도록 설계했다.

## 주요 구조
- `UCharacterAttributeSet`
  - `Stamina` / `MaxStamina` / `StaminaRegenRate`
- `GameplayTagSystem`
  - `Cooldown.StaminaRegen`
- `ABaseCharacter`
  - `DefaultEffects` (무한 회복 패시브 및 초기 스탯 세팅용)
  - `OnStaminaChanged` / `OnHealthChanged` (UI 델리게이트)
- `UGA_Sprint` (C++) / `BP_GA_Sprint` (BP)
  - `SprintEffectClass` / `SprintDrainEffectClass` / `SprintRegenPauseEffectClass`
- **Gameplay Effects (BP)**
  - `BP_GE_InitStats`, `BP_GE_StaminaRegen`, `BP_GE_StaminaCost`, `BP_GE_SprintStamina`, `BP_GE_StaminaRegenPause`

## 구현 파일
- `Source/TeamProject_Fin/GameCore/Attribute/CharacterAttributeSet.h`
- `Source/TeamProject_Fin/GameCore/Attribute/CharacterAttributeSet.cpp`
- `Source/TeamProject_Fin/GameCore/Tags/GameplayTagSystem.h`
- `Source/TeamProject_Fin/GameCore/Tags/GameplayTagSystem.cpp`
- `Source/TeamProject_Fin/GameCore/Character/BaseCharacter.h`
- `Source/TeamProject_Fin/GameCore/Character/BaseCharacter.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_Sprint.h`
- `Source/TeamProject_Fin/GameCore/Ability/GA_Sprint.cpp`

## 주요 구현 내용 및 시스템 역할

### C++ 코어 구현

#### UCharacterAttributeSet
- **역할**: 캐릭터의 스태미나 속성과 재생 속도를 관리한다.
- `StaminaRegenRate` 변수를 신규 추가하고 네트워크 동기화(Replication)를 구성했다. (기본값은 블루프린트 GE를 통해 덮어씌워짐)
- `PreAttributeChange` 및 `PostGameplayEffectExecute` 내에서 최소/최댓값 클램핑(Clamp)을 처리하여 값이 범위를 벗어나지 않도록 방어한다.

#### GameplayTagSystem
- **역할**: 스태미나 회복 제어를 담당할 전용 상태/쿨다운 태그를 정의한다.
- `Cooldown_StaminaRegen` 태그를 네이티브로 등록했다. 이후 블루프린트 단에서 무한 회복 이펙트가 이 태그를 감지하여 억제(Suppress)되도록 설계된다.

#### ABaseCharacter
- **역할**: 캐릭터의 초기화 및 컴포넌트 이벤트 래핑(UI 전송용)을 담당한다.
- **DefaultEffects 배열 추가**: 서버 권한에서 스폰될 때 캐릭터에게 기본 적용할 이펙트 리스트(`TArray<TSubclassOf<UGameplayEffect>>`)를 추가했다. 이를 통해 초기 스탯 세팅(`BP_GE_InitStats`)과 무한 스태미나 회복 패시브(`BP_GE_StaminaRegen`)를 엔진 시작과 동시에 자동 적용한다.
- **UI 연동 델리게이트**: `FOnStaminaChangedDelegate`를 `BlueprintAssignable`로 선언했다. `ASC`의 속성 변경 델리게이트를 C++에서 구독하고 있다가 값이 바뀔 때마다 델리게이트를 쏴주어, 위젯 블루프린트에서 틱 없이 프로그레스 바를 갱신할 수 있게 한다.

#### UGA_Sprint
- **역할**: 달리기 어빌리티 실행 중 지속적인 스태미나 소모 및 고갈 시 강제 취소 로직을 처리한다.
- **효과 분리 (슬롯화)**: 
  - `SprintDrainEffectClass`: 달리는 동안 스태미나를 깎고 회복을 방지하는 주기적 GE 슬롯을 추가했다.
  - `SprintRegenPauseEffectClass`: 달리기가 끝난 직후 회복을 지연시키는 GE 슬롯을 추가했다.
- **고갈 감지(Cancellation)**: `ActivateAbility` 시 스태미나 속성 변경에 콜백(`OnStaminaChanged`)을 바인딩하여, 값이 `0.0f` 이하가 되는 순간 `CancelAbility()`를 호출해 달리기를 강제로 종료시키고 걷기로 전환한다.

### 블루프린트 및 GAS 데이터 세팅 구현
무거운 틱(Tick) 연산이나 복잡한 블루프린트 노드 스크립팅 없이, 오직 GAS의 주기적 실행(Periodic) 속성과 태그 기반 조건(Tag Requirements)만을 사용하여 '스태미나 소모 후 일정 시간 회복 정지(Regen Pause)' 기믹을 최적화하여 완성했다.

#### 0. 캐릭터 초기 스탯 세팅 (BP_GE_InitStats)
- 캐릭터 스폰 시 기본 능력치를 C++ 하드코딩 대신 블루프린트에서 유연하게 덮어씌우기(Override) 위한 이펙트다.
- `ABaseCharacter`의 `DefaultEffects` 배열에 등록되어 게임 시작 시 1회 자동 적용된다.
- **모디파이어**: `MovementSpeed`와 `StaminaRegenRate` 어트리뷰트 값을 `Override` 방식으로 지정하여 게임 기획에 맞는 초기값으로 세팅한다.

#### 1. 기본 스태미나 재생 패시브 (BP_GE_StaminaRegen)
- 캐릭터가 스폰될 때 항시 적용받는 기본 회복 이펙트다.
- **기간 설정**: 경과시간 정책을 `Infinite`(무한)로 설정하고, 0.05초마다 주기적 이펙트가 실행되도록 세팅했다.
- **회복량**: 확장성을 위해 모디파이어를 `Attribute Based`로 설정했다.
  - **Backing Attribute**: `StaminaRegenRate`
  - **Coefficient (계수)**: `0.05`
  - 이를 통해 0.05초마다 초당 회복량의 5%씩 아주 부드럽게 채워주며, 향후 스태미나 회복 속도를 올리는 아이템/버프 추가 시 `StaminaRegenRate` 속성만 변경하면 이펙트 수정 없이 회복 속도가 즉각 연동된다.
- **핵심 기믹 (회복 차단)**: 타깃 태그(Target Tags)의 '태그가 없어야 함' 조건에 `Cooldown.StaminaRegen` 태그를 명시했다. 이를 통해 캐릭터가 구르기나 달리기로 인해 해당 쿨다운 태그를 부여받는 즉시 이 이펙트의 회복 효과가 시스템 레벨에서 자동 억제(Suppress)된다.

#### 2. 단발성 스태미나 소모 코스트 (BP_GE_StaminaCost)
- 구르기(Dodge)나 강공격 등 단발성 액션을 취할 때 즉발로 소모되는 스태미나 이펙트다.
- **기간 설정**: 경과시간 정책을 `Instant`(즉각적)으로 설정했다.
- **소모량**: 모디파이어를 통해 실행 즉시 `Stamina` 어트리뷰트를 -30.0만큼 차감시킨다.

#### 3. 지속성 스태미나 소모 및 회복 억제 (BP_GE_SprintStamina)
- 달리기(Sprint) 상태를 유지하는 동안 지속적으로 스태미나를 깎아내는 GE 슬롯이다.
- **기간 설정**: 0.05초 주기로 실행되는 주기적 이펙트로 구성했다.
- **소모량**: 한 주기마다 스태미나를 -1.0씩 지속 감소시킨다.
- **태그 부여**: 이펙트가 유지되는 동안 액터에게 `Cooldown.StaminaRegen` 태그를 부여(Granted Tags)한다. 이 태그 덕분에 달리는 도중에는 `BP_GE_StaminaRegen`의 회복 로직이 절대 발동하지 않는다.

#### 4. 액션 종료 후 회복 지연 타이머 (BP_GE_StaminaRegenPause)
- 달리기나 구르기 등 스태미나 소모 행위가 끝난 직후, 바로 스태미나가 차오르지 않게 만드는 딜레이 생성 이펙트다.
- **기간 설정**: 경과시간 정책을 `Has Duration`으로 설정하고, 규모를 0.8로 지정하여 정확히 0.8초 동안만 유지되도록 만들었다.
- **태그 부여**: 유지되는 0.8초간 `Cooldown.StaminaRegen` 태그를 액터에게 부여한다. 즉, 액션이 끝났어도 이 0.8초 동안은 회복 패시브가 여전히 차단된 상태로 남아있게 된다.

#### 5. 어빌리티 연동 설정 (BP_GA_Sprint)
- C++에서 노출시킨 이펙트 클래스 변수에 위에서 제작한 GE들을 할당하여 로직을 완성했다.
- **Sprint Drain Effect Class**: `BP_GE_SprintStamina`를 할당하여, 어빌리티 활성화 중 지속적인 스태미나 감소를 유발한다.
- **Sprint Regen Pause Effect Class**: `BP_GE_StaminaRegenPause`를 할당하여, 어빌리티 종료(Cancel 또는 자연 종료) 시 C++ 단에서 이 이펙트를 발동시켜 0.8초의 유예 시간을 발생시킨다.

## 관련 구현
- **#6 3인칭 캐릭터 이동 구현**: 캐릭터 달리기(`GA_Sprint`) 어빌리티에 스태미나 소비 및 회복 지연 로직(GE 슬롯화 및 스태미나 고갈 시 강제 취소)을 통합 연동했다.

## 사용 방법
- 신규 액션 어빌리티(예: 구르기, 공격 등)를 개발할 때, `Cost Gameplay 정Effect Class`에 `BP_GE_StaminaCost`를 넣는다.
- 어빌리티 시전 직후 노드에서 `Apply Gameplay Effect To Owner`를 호출하여 `BP_GE_StaminaRegenPause`를 적용해 주면, 모든 액션에 동일한 0.8초 회복 지연 룰을 손쉽게 적용할 수 있다.

## 테스트 또는 확인 내용
- [x] 빌드 성공 확인 (UE 5.8 Editor)
- [x] **초기 스탯 검증**: `BP_GE_InitStats`를 통해 덮어씌운 이동속도 및 `StaminaRegenRate` 값이 게임 진입 시 정상 반영되는지 확인 완료.
- [x] **기본 회복 테스트**: PIE 환경에서 아무런 조작을 하지 않을 때 `StaminaRegenRate * 0.05` 공식을 통해 스태미나가 0.05초마다 부드럽게 차오르는지 확인 완료.
- [x] **달리기 소모 및 차단 테스트**: Shift 키를 눌러 달리는 동안 스태미나가 깎이기만 할 뿐 동시에 회복되지 않음을 태그 시스템을 통해 검증 완료.
- [x] **딜레이(Regen Pause) 테스트**: 구르기를 시전하거나 달리기를 멈춘 후, 즉시 스태미나가 차지 않고 0.8초를 대기한 뒤에야 재생이 시작되는 하드코어 템포 동작 확인 완료.
- [x] 달리기(Sprint) 시 스태미나 0 도달 시 강제 걷기 전환 기능 동작 확인 완료.

## 남은 작업 또는 확장 방향
- C++에서 선언한 `OnStaminaChanged` 델리게이트를 수신하여 화면 좌측 상단에 스태미나 바(HUD)의 Fill 비례를 실시간으로 조절하는 위젯 블루프린트 연동 작업 진행 예정.
- 향후 데미지 및 체력 시스템, 그리고 무기 발사 어빌리티 추가 시 공통 Cost 시스템으로 확장 적용 가능성 검토.
