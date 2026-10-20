# Issue #11 구르기(회피) 모션 및 어빌리티 구현

## Issue
- **Issue 번호**: #11
- **기능명**: 구르기(회피) 모션, 방향별 제어 및 어빌리티 구현 (C++ 및 Blueprint 연동)
- **담당자: seok0317
- **담당 영역**: Ability System / Character / Tags / Blueprint / Animation Montage
- **관련 의존성**: #6 BaseCharacter (기본 캐릭터 구조), #10 스태미나 시스템 코어 구현

## 구현 목적
캐릭터가 스태미나를 소모하여 회피(구르기)를 수행할 수 있도록 C++ 코어 시스템과 블루프린트 어빌리티를 유기적으로 결합한다.
- **C++ 파트**: 회피 태그(`Ability.Movement.Dodge`, `State.Movement.Dodging`)와 스태미나 검증 로직을 구축한다.
- **블루프린트 파트**: 플레이어의 실제 조작(입력 벡터)에 반응하는 구르기 어빌리티(`BP_GA_Dodge`)를 구현한다. 8방향 입력에 따른 애니메이션 몽타주 섹션 분기, 카메라 멀미 방지(디커플링), 스태미나 지연 이펙트를 조립한다.

## 주요 구조
```text
 UAbilitySystemComponent (ASC)
  ├─ UGameplayTagSystem (C++ 전역 태그 식별자)
  │   ├─ Ability.Movement.Dodge
  │   ├─ State.Movement.Dodging
  │   └─ Data.StaminaCost
  └─ UCharacterAttributeSet (C++ 스태미나 스탯 관리)
      └─ MaxStamina, Stamina

 BP_GA_Dodge (블루프린트 어빌리티)
  ├─ 방향 판별 로직 (Input Vector 기반)
  ├─ Play Montage and Wait (AM_Roll 몽타주 재생)
  │   └─ Forward, Left, Right, ForwardLeft, ForwardRight 섹션 분기
  ├─ Camera Decoupling (구르기 중 카메라 랙 보정)
  └─ Cost 적용 (BP_GE_StaminaRegenPause 호출)
```

## 구현 파일
- **C++**:
  - `Source/TeamProject_Fin/GameCore/Tags/GameplayTagSystem.h/cpp`
  - `Source/TeamProject_Fin/GameCore/Character/BaseCharacter.h/cpp`
  - `Source/TeamProject_Fin/GameCore/Attribute/CharacterAttributeSet.h/cpp`
- **Blueprint/Asset**:
  - `BP_GA_Dodge` (회피 어빌리티 블루프린트)
  - `AM_Roll` (구르기 애니메이션 몽타주)
  - `UC_DodgeCurve` (구르기 이동력 제어 커브)

## 주요 구현 내용 및 시스템 역할

### 1. 전역 태그 및 스탯 연동 (C++ 코어)
- **GameplayTagSystem**: 회피 스킬 호출 식별자, 회피 동작 상태 식별자, 스태미나 소모 데이터 태그를 하드코딩하여 블루프린트에서 활용할 수 있도록 제공한다.
- **CharacterAttributeSet**: `Data_StaminaCost` 태그를 참조하여 현재 `Stamina` 어트리뷰트가 충분한지 검사하며, 부족 시 어빌리티 발동을 시스템 레벨에서 자동 차단한다.

### 2. 입력 기반 몽타주 섹션 분기 (Blueprint)
사용자가 누르고 있는 이동 키(WASD)에 맞춰 자연스러운 방향으로 구르도록 애니메이션 몽타주(`AM_Roll`)를 구성했다.
- **섹션(Section) 분할**: 하나의 몽타주 안에 Forward, Left, Right, ForwardLeft, ForwardRight 등 여러 방향의 구르기 모션을 섹션으로 나누어 배치했다.
- **동적 재생**: 어빌리티가 실행될 때 캐릭터의 Movement Input Vector를 계산하여, 알맞은 몽타주 섹션 이름을 `Play Montage and Wait` 노드에 전달해 해당 방향으로 부드럽게 구르도록 구현했다.

### 3. 어빌리티 태그 및 상호 배타성 세팅 (Blueprint)
구르기는 생존을 위한 최우선 액션이므로, 다른 모든 하위 액션들을 취소하거나 덮어쓰도록 GAS 태그를 설정했다.
- **에셋 태그**: `Ability.Movement.Dodge`를 부여하여 어빌리티의 정체성을 시스템에 알린다.
- **태그 포함 어빌리티 취소 (Cancel)**: 구르기를 시작하면 현재 진행 중인 달리기(Jogging), 앉기(Crouching), 조준(Aiming), 장전(Reloading) 어빌리티를 즉시 취소시킨다.
- **태그 있는 능력 차단 (Block)**: 구르기가 완전히 끝날 때까지 위 액션들의 재실행을 완벽히 차단한다.

### 4. 이동 보정 및 카메라 연출 (Blueprint)
단순한 루트 모션을 넘어, 커브 데이터와 카메라 튜닝을 통해 게임플레이 감각을 끌어올렸다.
- **이동력 제어**: Dodge Force Strength(800.0)와 Dodge Force Curve(`UC_DodgeCurve`)를 조합하여, 구르기 초반에 폭발적으로 거리를 벌리고 후반에 자연스럽게 감속하도록 커브 기반 이동을 세팅했다.
- **카메라 디커플링(Camera Lag)**: 
  - `Enable Dodge Camera Lag`를 활성화하고 Lag Speed 수치를 조정(7.0 / 10.0)했다.
  - 구르기 모션 중 캐릭터의 몸체가 심하게 회전할 때 카메라가 1:1로 쫓아가면 심각한 3D 멀미가 발생한다. 이를 방지하기 위해 구르기 시간 동안만 카메라의 추적 속도를 늦춰 화면을 안정적으로 유지한다.

### 5. 스태미나 코어 연동 (Blueprint)
- **Cost**: Dodge Regen Pause Effect Class 항목에 이슈 #10에서 제작한 `BP_GE_StaminaRegenPause`를 할당했다. 이로써 구르기 액션이 종료됨과 동시에 0.8초의 스태미나 회복 지연 타이머가 완벽하게 작동한다.

## 테스트 또는 확인 내용
- **방향별 구르기 테스트**: 이동 키(WASD) 조합에 따라 제자리, 전진, 대각선 등 의도한 섹션의 애니메이션이 정확히 분기되어 재생되는지 PIE에서 확인 완료.
- **카메라 안정성 검증**: 구르기 시 화면이 어지럽게 회전하지 않고, 부드럽게 캐릭터의 최종 위치로 따라붙는 카메라 랙(Lag) 연출 작동 확인.
- **액션 우선순위 검증**: 조준(우클릭) 또는 장전 중 구르기 키 입력 시, 조준/장전 모션이 즉각 캔슬되며 구르기로 이행하는지 태그 상호작용 확인 완료.
- **C++ 스탯 동기화**: AttributeSet의 Stamina 속성 소모 및 컴파일 정상 확인.
- **Result**: Succeeded

## 남은 작업 또는 확장 방향
- **무적 프레임(I-Frame) 연동**: 다음 이슈(#12 구르기 무적 판정 연동)를 통해, 몽타주 에디터 내에 `AnimNotifyState`를 배치하고, 해당 체공 구간 동안 캐릭터에게 `State.Invincible` 태그를 부여하여 완벽한 회피 판정을 완성할 예정이다.ate_Movement_Dodging` 태그가 부여되면, 추후 C++ 피격 로직이나 상태 머신에서 이 태그의 유무를 검사해 추가 로직(이동 제한 등)을 분기할 수 있다.

## 테스트 방법

### 에디터 설정 (Blueprint 병합 대기)
1. C++ 컴파일을 통해 추가된 `Ability.Movement.Dodge`, `State.Movement.Dodging` 태그가 에디터에 노출되는지 확인한다.
2. (Blueprint) `GA_Dodge` 어빌리티를 생성하고 Ability Tags에 해당 식별자를 부여한다.
3. (Blueprint) 플레이어 컨트롤러 또는 캐릭터 입력 이벤트(Enhanced Input)에서 해당 어빌리티를 트리거하도록 연결한다.

### 확인 결과 (C++ 환경)
- C++ 코드 단에서 정의한 매크로 태그가 에디터의 GameplayTag 딕셔너리에 정상적으로 매핑 및 로드됨을 확인.
- AttributeSet의 Stamina 속성이 정상적으로 컴파일 및 노출됨을 확인.
- 언리얼 엔진 5.8 Development Editor 빌드 성공.

- **Result**: Succeeded

## 완료 조건 확인 (C++ 파트)
- [x] 회피 어빌리티 발동용 Ability Tag 선언 및 등록
- [x] 회피 상태 판별용 State Tag 선언 및 등록
- [x] 스태미나 비용 처리용 Data Tag 선언 및 등록
- [x] 빌드 성공 확인

## 남은 작업 (Blueprint 파트와 병합 필요)
다음 기능은 블루프린트 기술 문서와 합쳐져야 완벽한 기능 설명이 완성된다.

- **GA_Dodge (블루프린트)** 로직 구현:
  - Input Vector를 기반으로 전/후/좌/우 구르기 분기 처리 (BlendSpace 또는 분기 노드).
  - 루트 모션 애니메이션 몽타주 `Play Montage and Wait` 적용.
  - 구르기 중 카메라 회전 동기화 해제(Decoupling) 로직.
  - 구르기 시작 시 이동 제어권 잠금 및 몽타주 종료 시 이동 권한 복구 로직.

## 확장 방향
- 다음 이슈(#12 구르기 무적 판정 연동)에서 `AnimNotifyState`와 C++ 데미지 시스템(`ExecCalc_Damage`)을 연결하여, 회피 모션 중 체공 시간 동안 `State_Invincible` 무적 태그를 부여하는 형태로 확장된다.
