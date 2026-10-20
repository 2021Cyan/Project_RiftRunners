# #4 GameplayTag 구조 정의

## Issue

- Issue 번호: #4
- 기능명: GameplayTag 구조 정의
- 담당자: 2021Cyan
- 관련 Sprint: 1주차

## 목적

전투, 스킬, 데미지, 쿨다운, 상태 처리를 문자열 하드코딩 없이 공통 GameplayTag로 관리합니다.

## 구현 내용

- Native GameplayTag 선언/정의 구조 추가
- `GameplayTagSystem` namespace로 태그 접근 방식 통일
- Character / State / Ability / Event / Damage / Cooldown / Data / Effect / Weapon 카테고리 구성
- 데미지 출처와 타입 구분 태그 추가
- 메인 스킬 쿨다운 태그 추가
- SetByCaller용 Data 태그 추가
- 버프/디버프 및 환경 효과 확장용 태그 추가

## 관련 구현

- C++: `GameplayTagSystem`
- Blueprint:
- Asset:

## 사용 방법

태그를 사용하는 C++ 파일에서 GameplayTagSystem 헤더를 include합니다.

```cpp
#include "../Tags/GameplayTagSystem.h"
```

예시: 메인 스킬 쿨다운 상태 확인

```cpp
const bool bOnCooldown =
    ASC->HasMatchingGameplayTag(GameplayTagSystem::Cooldown_Skill_Main);
```

예시: 데미지 타입 태그 사용

```cpp
FGameplayTag DamageType = GameplayTagSystem::Damage_Type_Weapon;
```

## 테스트 및 확인 내용

- [x] Native GameplayTag 선언/정의 컴파일 확인
- [x] 쿨다운 태그 적용 및 제거 확인
- [x] Damage/Data 태그를 DamageSystem과 GameplayAbility에서 사용 확인
- [x] 관련 Issue 완료 조건 확인

## 남은 작업

- 없음

## 확장 방향

- 스킬이 늘어나면 `Ability.*`, `Cooldown.*` 태그를 추가합니다.
- 상태 이상이 늘어나면 `Effect.Buff.*`, `Effect.Debuff.*` 하위 태그를 확장합니다.
- 환경 효과가 늘어나면 `Effect.Environment.*` 하위 태그를 확장합니다.
