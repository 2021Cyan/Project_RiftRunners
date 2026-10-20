# #3 AttributeSet 구현

## Issue

- Issue 번호: #3
- 기능명: AttributeSet 구현
- 담당자: 2021Cyan
- 관련 Sprint: 1주차

## 목적

GAS 기반 캐릭터 능력치를 한 곳에서 관리하고, 데미지/회복/스태미나 소모 같은 GameplayEffect 적용 결과가 안정적으로 반영되도록 합니다.

## 구현 내용

- `UCharacterAttributeSet` 추가
- `Health`, `MaxHealth`, `Stamina`, `MaxStamina`, `MovementSpeed` Attribute 구성
- Attribute Replication / RepNotify 처리
- `Health`, `Stamina` 최소/최대값 Clamp 처리
- `MaxHealth`, `MaxStamina` 변경 시 현재값 비율 보정
- `MovementSpeed` 변경 시 `CharacterMovement.MaxWalkSpeed` 반영
- `ABaseCharacter`에 `AbilitySystemComponent`와 `CharacterAttributeSet` 연결

## 관련 구현

- C++: `CharacterAttributeSet`, `BaseCharacter`
- Blueprint:
- Asset:

## 사용 방법

캐릭터에서 AttributeSet에 접근해 현재 Attribute 값을 확인할 수 있습니다.

```cpp
UCharacterAttributeSet* AttributeSet = BaseCharacter->GetCharacterAttributeSet();

float Health = AttributeSet->GetHealth();
float Stamina = AttributeSet->GetStamina();
```

GameplayEffect 또는 ASC Modifier로 Attribute가 변경되면 `PostGameplayEffectExecute`에서 최종 Clamp가 적용됩니다.

## 테스트 및 확인 내용

- [x] 데미지 적용 시 `Health` 감소 확인
- [x] 회복 적용 시 `Health` 증가 확인
- [x] 스태미나 소모 시 `Stamina` 감소 확인
- [x] `Health`, `Stamina`가 0 아래로 내려가지 않음
- [x] `Health`, `Stamina`가 최대값을 초과하지 않음
- [x] 관련 Issue 완료 조건 확인

## 남은 작업

- 없음

## 확장 방향

- 추후 방어력과 같은 추가 전투 Attribute를 추가할 수 있습니다.
- HUD 연동 시 Attribute Change Delegate를 통해 UI를 갱신합니다.
