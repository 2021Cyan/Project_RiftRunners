# #35 적 체력, 피격 및 사망 처리 구현

## Issue

- Issue 번호: #35
- 기능명: 적 체력, 피격 및 사망 처리 구현
- 담당자: horimihorimi
- 담당 영역: Enemy Combat / GAS / Attribute / State Tree / Pooling
- 선행 이슈: #13 체력 및 데미지 처리 구현, #14 피격 반응 및 그로기 시스템 구현

## 목적

일반 몬스터와 보스가 공통 피해 경로를 사용하도록 구성하고 체력 변경, 피격 반응, 사망 시작·완료를 분리한다. 사망 중복 처리와 풀 재사용 시 이전 상태가 남는 문제를 방지한다.

## 주요 구조

```text
Definition 스탯 행 → Character 초기 스탯 적용
    ↓
Damage Gameplay Effect → Attribute 변경
    ↓
CombatDamageEventRouter
├─ 실제 적용 피해 / 전투 통계 / 보상 기여도
├─ 피격 알림 → HitReact 이벤트 및 GA
└─ 체력 0 이하 → Event.Death → GA_Death
    ↓
BeginDeath → AI 중단 / 이동 정지 / 충돌 응답 변경
    ↓
사망 몽타주 종료 또는 중단 → FinishDeath
    ↓
OnMonsterDeath → 보상 연결 / 시체 유지 / 제거 또는 풀 반환
```

## 설계 결정

### 스탯 데이터와 반응 흐름 분리

Definition은 StatTableRowHandle로 초기 스탯을 참조한다. CommonEnemyCharacter는 기존 초기화 후 Definition의 스탯 행을 적용한다. 피격·회피 반응과 사망 흐름은 Definition의 별도 Reaction Set이 아니라 Character, State Tree 및 GA가 처리한다.

### 실제 적용 피해 사용

피해 통계와 보상 기여도는 AppliedDamage를 사용한다. 사망 이벤트를 보내기 전에 마지막 타격을 기록한다. 보상 정책은 [#36 적 처치 보상](./Issue-036-Enemy-Drop-Reward.md), 보스 클리어는 [#40](./Issue-040-Boss-Death-Game-Clear.md)을 참고한다.

## 구현 내용

### 체력과 표시 데이터

CharacterAttributeSet이 체력과 최대 체력을 관리한다. 초기 스탯은 스탯 테이블에서 설정하며 몬스터·보스 UI의 표시 이름은 Enemy Definition의 DisplayName을 사용한다. 개별 체력·피해 수치는 테이블과 관련 에셋에서 조정한다.

### 피격 반응

MonsterCharacter는 누적 피해와 피격 횟수 등을 관리하고 HitReact 이벤트를 ASC에 전달한다. 몬스터 피격 경로는 일반 HitReact를 사용하며 Heavy HitReact와 넉백을 사용하지 않는다. GA와 상태 태그가 반응 실행 및 전투 행동 제한을 처리한다.

### 사망 시작

새 체력이 0 이하가 되는 순간을 사망으로 판정한다. Event.Death로 실행된 GA_Death가 BeginDeath를 호출하고 다른 Ability를 취소한다. Character는 사망 시작 상태를 설정하고 OnDeathStarted를 전달한다. MonsterCharacter는 컨트롤러의 StopAIForDeath로 AI를 중단한다.

### 사망 완료

사망 몽타주 완료·중단·취소 시 FinishDeath 경로를 처리한다. 몽타주가 없어도 완료한다. bDeathStarted와 bDeathFinished로 중복 실행을 방지한다. 서버의 MonsterCharacter는 OnMonsterDeath를 전달한다.

### 시체와 풀 재사용

일반 몬스터는 CorpseLifeSpan 설정에 따라 제거한다. 풀링 몬스터는 즉시 파괴하지 않고 스폰 포인트가 시체 유지 후 풀에 반환한다. 재활성화 경로에서는 이전 몽타주와 사망 상태, 초기 스탯, 이동 프로필 및 특성을 정리·재적용한다. 반환 및 재사용의 상세 실행 순서는 풀·스폰 경로와 함께 검증한다.

## 관련 구현

### C++

- `Source/TeamProject_Fin/GameCore/Attribute/CharacterAttributeSet.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Attribute/MonsterInitData.h`
- `Source/TeamProject_Fin/GameCore/Character/BaseCharacter.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Character/MonsterCharacter.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Character/CommonEnemyCharacter.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_Death.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Combat/CombatDamageEventRouter.cpp`
- `Source/TeamProject_Fin/GameCore/Enemy/Spawn/EnemyPoolSubsystem.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Enemy/Spawn/SpawnPoint.h/.cpp`
- `Source/TeamProject_Fin/GameCore/UI/MonsterStatusWidget.h/.cpp`
- `Source/TeamProject_Fin/GameCore/UI/BossBattleHUDWidget.h/.cpp`

### Blueprint / Asset

- `Content/Monster/DT/DT_MonsterInitState`
- `Content/Monster/UI/WBP_MonsterStatus`
- `Content/Monster/UI/WBP_BossBattleHUD`
- `Content/Monster/DA/Melee/DA_Melee_Definition`
- `Content/Monster/DA/Range/DA_Range_Definition`
- `Content/Monster/DA/Elite/DA_Elite_Definition`
- `Content/Monster/DA/Boss/DA_Boss_EnemyDefinition`

## 사용 방법

1. Definition의 StatTableRowHandle에 몬스터 초기 스탯 행을 지정한다.
2. 표시 이름, 기본 Ability와 종류별 Ability Set을 구성한다.
3. 피격 및 사망 몽타주와 관련 GA·Effect를 연결한다.
4. 배치별 시체 유지 및 풀 사용 설정을 지정한다.
5. 체력 UI 및 사망 후 이벤트 연결을 확인한다.

## 테스트 및 확인 항목

개별 거리·확률·피해량·쿨다운 수치는 DA, BP 및 몽타주 설정에서 관리한다. 이 문서의 확인 항목은 실행 검증 체크리스트이며 완료 결과를 의미하지 않는다. Blueprint 내부 참조와 ST의 실제 전이는 에디터에서 확인한다.

- [ ] Definition 스탯 행과 체력 UI 초기화 확인
- [ ] 실제 적용 피해 및 마지막 타격 집계 확인
- [ ] 피격 누적과 상태 태그 해제 확인
- [ ] 사망 시 AI·이동·공격 중단 확인
- [ ] 사망 시작·완료 및 보상 알림 중복 방지 확인
- [ ] 사망 몽타주 누락·중단 시 완료 확인
- [ ] 일반 시체 제거와 풀 반환 확인
- [ ] 재사용 시 사망·피격·특성 상태 초기화 확인

## 후속 작업

- 몬스터별 피격·사망 에셋 연결 검증
- 멀티플레이 및 풀 재사용 반복 검증
