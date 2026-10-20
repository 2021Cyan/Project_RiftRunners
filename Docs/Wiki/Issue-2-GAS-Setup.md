# #2 Gameplay Ability System 기본 세팅

## Issue

- Issue 번호: #2
- 기능명: Gameplay Ability System 기본 세팅
- 담당자: 2021Cyan
- 관련 Sprint: 1주차

## 목적

캐릭터 능력치, GameplayEffect, GameplayTag, GameplayAbility를 프로젝트에서 사용할 수 있도록 GAS 기반 구조를 준비합니다.

## 구현 내용

- `GameplayAbilities` 플러그인 활성화
- `TeamProject_Fin.Build.cs`에 GAS 관련 모듈 의존성 추가
  - `GameplayAbilities`
  - `GameplayTags`
  - `GameplayTasks`
- `ABaseCharacter`가 `IAbilitySystemInterface`를 구현하도록 구성
- `UAbilitySystemComponent` 생성 및 Replication 설정
- `UCharacterAttributeSet`을 BaseCharacter에 연결
- BeginPlay에서 `InitAbilityActorInfo` 호출
- MovementSpeed Attribute 변경 Delegate 바인딩

## 관련 구현

- C++: `BaseCharacter`, `CharacterAttributeSet`
- Blueprint:
- Asset:

## 사용 방법

GAS를 사용하는 캐릭터는 `GetAbilitySystemComponent()`를 통해 ASC에 접근합니다.

```cpp
UAbilitySystemComponent* ASC = BaseCharacter->GetAbilitySystemComponent();
```

AttributeSet은 BaseCharacter에서 직접 가져올 수 있습니다.

```cpp
UCharacterAttributeSet* AttributeSet = BaseCharacter->GetCharacterAttributeSet();
```

GameplayEffect, GameplayTag, Attribute 관련 기능은 이 ASC와 AttributeSet을 기준으로 확장합니다.

## 테스트 및 확인 내용

- [x] `GameplayAbilities` 플러그인 활성화 확인
- [x] GAS 관련 Build.cs 모듈 의존성 확인
- [x] BaseCharacter ASC 생성 확인
- [x] AttributeSet 연결 확인
- [x] 관련 Issue 완료 조건 확인

## 남은 작업

- 없음

## 확장 방향

- GameplayAbility 기반 스킬 시스템을 추가합니다.
- 캐릭터 초기 능력 부여 로직을 추가합니다.
- 서버 권한 기준으로 Ability 실행과 GameplayEffect 적용 흐름을 확장합니다.
