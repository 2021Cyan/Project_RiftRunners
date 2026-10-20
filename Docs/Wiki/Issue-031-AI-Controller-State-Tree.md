# #31 AI Controller 및 State Tree 기본 구조 구현

## Issue

- Issue 번호: #31
- 기능명: AI Controller 및 State Tree 기본 구조 구현
- 담당자: horimihorimi
- 담당 영역: Enemy AI / State Tree / GAS / Data Asset
- 관련 Sprint: Sprint 2

## 목적

인지와 타깃 선택, 행동 상태 전환, 공격 실행을 분리한다. 일반 몬스터는 공용 State Tree를 사용하고 종류별 전투 차이는 Definition, Pattern Set, Ability Set 및 이동 프로필로 구성한다. 보스는 공용 패턴 실행 구조 위에 페이즈 전환과 아레나 이동을 추가한다.

## 주요 구조

```text
AI Perception / 조우 요청
    ↓
MonsterAIController → CommonEnemyAIController / BossEnemyAIController
    ↓
State Tree
├─ ST_Enemy_Common: 일반 몬스터 공용 흐름
└─ ST_Enemy_Boss: 보스 전투 및 페이즈 전환
    ↓
Select Enemy Pattern → Prepare Enemy Pattern
    ↓
Execute Enemy Pattern → Wait Pattern Recovery → 기본 이동
    ↓
Gameplay Ability / 몽타주 Notify / 서버 피해 적용
```

```text
Enemy Definition
├─ DisplayName / EnemyId
├─ StatTableRowHandle
├─ AbilitySet → AbilityTag / AbilityClass
├─ PatternSet → 실행 조건 / 이동 후보 / Recovery / InterruptPolicy
├─ MovementProfile → 이동 방식별 설정
└─ Emerge → 바닥·벽·천장 기본 등장 몽타주

캐릭터·스폰 배치
└─ 시작 방식 / 순찰 / 타깃 상실 대응 / 시체 유지 설정
```

## 설계 결정

### 전투 데이터와 상태 전환 분리

공격 후보를 몬스터별 State Tree에 복제하지 않는다. `UEnemyPatternComponent`가 활성 Pattern Set, 현재 패턴, 다음 패턴과 실행 상태를 관리한다. 거리·고도·체력·쿨다운·연속 사용 제한을 통과한 패턴을 가중치로 선택한다. 개별 수치는 DA에서 조정한다.

기존 `FSTT_ChooseWeightedAction` 등 레거시 클래스가 코드에 남아 있어도 일반 전투 패턴의 데이터 소유 위치는 Pattern Set이다. 클래스 존재만으로 현재 ST의 사용 여부를 판단하지 않는다.

### AI와 Gameplay Ability의 역할

컨트롤러는 인지, 타깃 평가와 일반 추격을 담당한다. State Tree는 행동 순서와 상태 전환을 담당한다. GA는 몽타주, 특수 이동, Gameplay Effect와 취소 시 정리를 담당한다.

일반 추격은 `State.AI.ActionCommitted`에 따라 제한된다. 보스 컨트롤러는 일반 추격 대신 보스 이동 경로를 사용한다.

## 구현 내용

### 인지와 타깃 선택

`AMonsterAIController`가 인지 정보와 타깃 평가를 관리한다. `ACommonEnemyAIController`는 휴면 중 평가를 중단하고 사망·은신 등 유효하지 않은 플레이어를 후보에서 제외한다. 타깃 선택의 상세 정책은 [몬스터 타깃 선택](./Issue-105-Monster-Target-Selection.md)을 참고한다.

### 패턴 선택과 준비

- `FSTT_SelectEnemyPattern`: 실행 가능한 패턴 선택
- `FSTT_PrepareEnemyPattern`: 선택된 이동 후보에 따라 위치 준비
- `RepositionForAction`: 위치 보정 컴포넌트가 Ability 태그로 프로필을 조회하고 재배치
- `bMoveDuringAbility`: 준비 완료를 기다리지 않고 GA와 이동을 병행
- `ANS_EnemyAttackPositioning`: 몽타주 구간의 공격 위치 이동 제어

### 패턴 실행과 회복

`FSTT_ExecuteEnemyPattern`은 실행 직전 조건을 다시 검사하고 Ability 태그로 GA를 활성화한다. 공격 토큰이 필요한 패턴은 전투 디렉터의 토큰을 확보해야 한다. 활성 Spec Handle을 추적해 종료를 확인하며, 상태 이탈 시 `InterruptPolicy`에 따라 취소 여부를 결정한다.

`FSTT_WaitPatternRecovery`는 패턴의 Recovery 범위에서 대기 시간을 정한다. `FSTT_EnemyIdleMovement`는 패턴 사이 기본 움직임을 처리한다.

### 등장·순찰·타깃 상실

시작 방식과 순찰 경로는 배치 설정으로 관리한다. 휴면 조우는 등장 GA를 거쳐 전투로 이어진다. 타깃 상실 대응은 탐색 또는 휴면 복귀로 구분하며, 순찰 경로·영역 확인과 탐색·복귀는 전용 Task 및 Condition으로 처리한다.

### 피격과 사망

Evaluator가 전투 상태와 태그를 State Tree에 전달한다. 피격 반응은 GA와 상태 태그로 연결한다. 사망 시 컨트롤러의 `StopAIForDeath()`로 AI를 중단한다. 상세 흐름은 [#35 적 체력·피격·사망](./Issue-035-Enemy-Health-Hit-Death.md)을 참고한다.

## 관련 구현

### C++

- `Source/TeamProject_Fin/GameCore/AI/MonsterAIController.h/.cpp`
- `Source/TeamProject_Fin/GameCore/AI/CommonEnemyAIController.h/.cpp`
- `Source/TeamProject_Fin/GameCore/AI/BossEnemyAIController.h/.cpp`
- `Source/TeamProject_Fin/GameCore/AI/MonsterTargetSelector.h/.cpp`
- `Source/TeamProject_Fin/GameCore/AI/MonsterCombatDirectorSubsystem.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Enemy/Data/EnemyDataAssets.h`
- `Source/TeamProject_Fin/GameCore/Combat/EnemyPatternComponent.h/.cpp`
- `Source/TeamProject_Fin/FSTE_SentinelPerception.h/.cpp`
- `Source/TeamProject_Fin/FSTE_MonsterCombatState.h/.cpp`
- `Source/TeamProject_Fin/FSTT_SelectEnemyPattern.h/.cpp`
- `Source/TeamProject_Fin/FSTT_PrepareEnemyPattern.h/.cpp`
- `Source/TeamProject_Fin/FSTT_ExecuteEnemyPattern.h/.cpp`
- `Source/TeamProject_Fin/FSTT_WaitPatternRecovery.h/.cpp`
- `Source/TeamProject_Fin/FSTT_EnemyIdleMovement.h/.cpp`
- `Source/TeamProject_Fin/FSTT_EnemyPatrol.h/.cpp`
- `Source/TeamProject_Fin/FSTT_EnemySearch.h/.cpp`
- `Source/TeamProject_Fin/FSTT_EnemyReturnToDormant.h/.cpp`

### Blueprint / Asset

- `Content/Monster/AI/ST/ST_Enemy_Common`
- `Content/Monster/AI/ST/ST_Enemy_Boss`
- `Content/Monster/AI/Controller/BP_MonsterAIController_Melee`
- `Content/Monster/AI/Controller/BP_MonsterAIController_Range`
- `Content/Monster/AI/Controller/BP_MonsterAIController_Elite`
- `Content/Monster/AI/BP_BossAIController`
- `Content/Monster/DA/Melee`
- `Content/Monster/DA/Range`
- `Content/Monster/DA/Elite`
- `Content/Monster/DA/Boss`

## 사용 방법

1. 이동 방식에 맞는 Enemy Character와 AI Controller를 지정한다.
2. 일반 몬스터는 ST_Enemy_Common, 보스는 ST_Enemy_Boss를 설정한다.
3. Enemy Definition에 스탯 행, Ability Set, Pattern Set, 이동 프로필과 등장 몽타주를 연결한다.
4. Pattern의 AbilityTag와 지급할 GA의 실행 태그를 맞춘다.
5. 패턴별 실행 조건, 이동 후보, Recovery 및 InterruptPolicy를 설정한다.
6. 스폰 배치의 시작 방식, 순찰 및 타깃 상실 대응을 설정한다.

## 테스트 및 확인 항목

개별 거리·확률·피해량·쿨다운 수치는 DA, BP 및 몽타주 설정에서 관리한다. 이 문서의 확인 항목은 실행 검증 체크리스트이며 완료 결과를 의미하지 않는다. Blueprint 내부 참조와 ST의 실제 전이는 에디터에서 확인한다.

- [ ] 일반 몬스터 종류별 Definition이 공용 패턴 흐름에서 적용되는지 확인
- [ ] 휴면·등장·인지·전투·타깃 상실 흐름 확인
- [ ] 실행 직전 조건 변경과 GA 활성화 실패 처리 확인
- [ ] 이동 준비 및 GA 병행 이동 확인
- [ ] 인터럽트 정책과 공격 토큰 반환 확인
- [ ] 사망 시 AI와 이동이 중단되는지 확인

## 후속 작업

- 실제 ST 노드 바인딩과 전이 우선순위는 에디터에서 검증
- 레거시 Task의 실제 참조 여부 확인 후 정리
- 멀티플레이에서 서버 AI 판정과 클라이언트 연출 통합 검증
