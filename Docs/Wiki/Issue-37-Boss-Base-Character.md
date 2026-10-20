# #37 보스 기본 캐릭터 구현

## Issue

- Issue 번호: #37
- 기능명: 보스 기본 캐릭터 구현
- 담당자: horimihorimi
- 담당 영역: Enemy Framework / Boss AI / GAS / Animation / Movement
- 관련 Sprint: Sprint 5
- 선행 이슈: #31 AI Controller 및 State Tree 기본 구조 구현, #35 적 체력·피격 및 사망 처리 구현

## 목적

Alepsis-Taura를 시작으로 최종 보스에 사용할 캐릭터 전투 구조를 구현한다.

보스 전용 로직을 기존 일반 몬스터에 직접 추가하지 않고, 공용 몬스터 전투 프레임워크 위에 보스 Character, 보스 이동 컴포넌트, 보스 Definition Data Asset을 조립하는 방식으로 구성한다. 이를 통해 보스마다 스탯, Ability Set, Pattern Set, 이동 프로필, 페이즈 구성을 데이터로 교체할 수 있도록 한다.

## 주요 구조

```text
ABaseCharacter
└─ AMonsterCharacter
   └─ ACommonEnemyCharacter
      └─ ABossMonsterCharacter
```

```text
BP_BossMonster
 └─ DA_Boss_EnemyDefinition
     ├─ StatTableRowHandle
     ├─ AbilitySet
     ├─ MovementProfile
     └─ BossPhaseSet
```

| 클래스 | 역할 |
|---|---|
| `ABaseCharacter` | GAS, 기본 스탯, Default Ability 및 Effect 초기화 |
| `AMonsterCharacter` | 조우, 사망, AI Controller 연동 |
| `ACommonEnemyCharacter` | Definition DA 기반 스탯, Ability Set, Pattern Set 구성 |
| `ABossMonsterCharacter` | 보스 페이즈, Form, 아레나 기준 이동 관리 |
| `UEnemyPatternComponent` | 현재 패턴, 다음 패턴, 활성 Pattern Set 관리 |
| `UBossMovementComponent` | 아레나 앵커 기준 부유, 공전, 재배치 이동 관리 |

## 설계 결정

### 공용 패턴 프레임워크 도입 배경

기존 State Tree는 `Choose Weighted Action` Task 내부에 몬스터별 공격 패턴 배열과 분기 조건을 직접 구성했다. 따라서 공격 패턴이 추가되거나, 회피·피격·강제 이벤트 등 몬스터별 반응 조건이 달라질 때마다 State Tree 에셋의 행동 후보 데이터와 전이 흐름을 수정해야 했다.

- 패턴 추가가 State Tree의 후보 데이터 수정으로 이어진다.
- 몬스터별 전투 데이터를 State Tree, Ability Set, Gameplay Ability를 오가며 확인해야 한다.
- 일반 몬스터와 보스가 같은 전투 흐름을 사용해도 몬스터마다 State Tree를 복제할 가능성이 높다.
- 보스의 페이즈별 패턴 교체를 State Tree 분기만으로 관리하면 전환 조건과 전투 데이터가 섞인다.

이를 해결하기 위해 전투 흐름은 공용 State Tree로 고정하고, 몬스터별 차이는 Enemy Definition 중심의 Data Asset으로 분리했다.

```text
패턴 선택
→ 사전 이동
→ Gameplay Ability 실행
→ Recovery 대기
→ 일반 추적 복귀
```

`Recovery`는 별도 이동 Task가 아니다. Gameplay Ability가 종료되어 `State.AI.ActionCommitted` 태그가 해제되면, 기존 AI Controller의 CMC 기반 일반 추적 이동이 다시 이어진다.

### 데이터 중심 전투 구성

몬스터별 전투 데이터는 Enemy Definition에서 관리한다.

- Pattern Id / Ability Tag
- 쿨다운, 거리 조건, 체력 조건, 가중치
- 패턴별 사전 이동 옵션
- Ability Set 및 이동 Profile 참조
- 보스 페이즈별 Pattern Set

```text
Enemy Definition
├─ Stat Table Row
├─ Ability Set
├─ Movement Profile
├─ Pattern Set
└─ Boss Phase Set
   ├─ Phase 1 Pattern Set
   ├─ Phase 2 Pattern Set
   └─ Phase 3 Pattern Set
```

새 몬스터 또는 새 패턴을 추가할 때는 원칙적으로 공용 State Tree를 수정하지 않는다.

1. Enemy Definition에 Pattern Set을 구성한다.
2. Ability Set에 Pattern GA를 등록한다.
3. 해당 Pattern GA와 필요한 몽타주, 노티파이, Effect를 추가한다.

전용 State Tree는 등장 연출, 강제 컷신, 특수 기믹처럼 공용 Combat Loop만으로 표현하기 어려운 경우에만 사용한다.

### 보스 기본 구조

보스는 공용 프레임워크를 사용하면서, 보스 고유 요소만 별도로 추가한다.

```text
ACommonEnemyCharacter
└─ ABossMonsterCharacter
   ├─ UBossPhaseComponent
   ├─ UBossMovementComponent
   └─ Boss Phase Set
```

- `ABossMonsterCharacter`는 보스 페이즈, Form, 아레나 기준 이동을 담당한다.
- `UBossPhaseComponent`는 체력 비율과 현재 페이즈 상태를 관리한다.
- `UBossMovementComponent`는 부유, 공전, 전환 재배치 같은 보스 전용 이동을 담당한다.
- 페이즈별 공격 차이는 State Tree를 복제하지 않고 Pattern Set 교체로 처리한다.

## 구현 내용

### 공용 몬스터 Definition 구성

- `UEnemyDefinitionDataAsset`에 스탯 행, Ability Set, Pattern Set, 이동 프로필을 정의한다.
- `ACommonEnemyCharacter`는 기존 BP `InitStatsRow` 초기화 후 Definition DA의 스탯 행을 추가 적용한다.
- 기존 `MonsterCharacter` 계열의 초기화 방식은 유지한다.
- Ability Set에 등록된 GA를 서버에서 지급한다.
- `UEnemyPatternComponent`가 활성 Pattern Set과 패턴 실행 상태를 보관한다.

### 보스 Character 및 이동

- `ABossMonsterCharacter`는 `Character.Enemy.Boss` 역할 태그를 사용하고 `UBossPhaseComponent`를 보유한다.
- 레벨에서 `BossArenaCenter` 태그를 가진 액터를 찾아 보스 이동의 기준점으로 사용한다.
- `UBossMovementComponent`는 앵커 기준 부유, 공전, 전환 중 재배치 및 목표 위치 이동을 지원한다.
- 패턴 실행 중에는 `State.AI.ActionCommitted`를 기준으로 일반 추적 이동과 패턴 이동이 동시에 적용되지 않게 한다.

### 보스 AI 및 애니메이션

- `BP_BossAIController`와 `ST_Enemy_Boss`를 구성했다.
- `ABP_Boss`는 Local Velocity를 기반으로 Strafe Blend Space를 재생한다.
- `BP_BossMonster`에 Alepsis-Taura Skeletal Mesh, Anim BP, AI Controller, Definition DA를 연결한다.

## 관련 구현

### C++

- `Source/TeamProject_Fin/GameCore/Character/CommonEnemyCharacter.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Character/BossMonsterCharacter.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Combat/EnemyPatternComponent.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Movement/BossMovementComponent.h/.cpp`
- `Source/TeamProject_Fin/GameCore/AI/CommonEnemyAIController.h/.cpp`
- `Source/TeamProject_Fin/GameCore/AI/BossEnemyAIController.h/.cpp`

### Blueprint / Asset

- `Content/Monster/BP/BP_BossMonster`
- `Content/Monster/AI/BP_BossAIController`
- `Content/Monster/AI/ST/ST_Enemy_Boss`
- `Content/Monster/Anim/ABP_Boss`
- `Content/Monster/DA/Boss/DA_Boss_EnemyDefinition`
- `Content/Monster/DA/Boss/DA_Boss_AbilitySet`
- `Content/Monster/DA/Boss/DA_Boss_MovementProfile`

## 사용 방법

1. `ABossMonsterCharacter` 기반 보스 Blueprint를 생성한다.
2. Skeletal Mesh, Anim BP, AI Controller를 지정한다.
3. 보스 Definition DA에 스탯 행, Ability Set, 기본 이동 프로필, Boss Phase Set을 지정한다.
4. 레벨에 `BossArenaCenter` 태그를 가진 기준 액터를 배치한다.
5. 보스 AI Controller에 `ST_Enemy_Boss`를 지정한다.
6. Pattern Set에 사용할 공격 GA를 Ability Set에 등록한다.

## 테스트 및 확인 항목

개별 거리·확률·피해량·쿨다운 수치는 DA, BP 및 몽타주 설정에서 관리한다. 이 문서의 확인 항목은 실행 검증 체크리스트이며 완료 결과를 의미하지 않는다. Blueprint 내부 참조와 ST의 실제 전이는 에디터에서 확인한다.

- [ ] Definition DA의 스탯 행이 보스 초기 스탯에 적용되는지 확인
- [ ] 보스가 조우 후 플레이어를 타깃으로 설정하는지 확인
- [ ] 보스가 기본 부유 및 Strafe 이동을 수행하는지 확인
- [ ] Pattern Set의 Ability가 실행되는지 확인
- [ ] 패턴 실행 중 일반 추적 이동이 중단되는지 확인
- [ ] 아레나 중심 기준 이동이 정상 동작하는지 확인
- [ ] 보스 사망 시 AI 및 전투 행동이 중단되는지 확인

## 후속 작업

- [#40 보스 사망 및 게임 클리어 처리](Issue-40-Boss-Death-Game-Clear)의 멀티플레이 통합 검증
- 보스 전용 체력바의 표시 이름 및 상태 갱신 검증
- 패턴별 VFX, SFX 및 Gameplay Cue 연동
- 멀티플레이 환경에서 보스 이동 및 Form 동기화 검증
