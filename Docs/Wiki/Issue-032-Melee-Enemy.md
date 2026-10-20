# #32 근접형 일반 적 구현

## Issue

- Issue 번호: #32
- 기능명: 근접형 일반 적 구현
- 담당자: horimihorimi
- 담당 영역: Enemy AI / GAS / Animation / Combat
- 관련 Sprint: Sprint 2
- 선행 이슈: #31 AI Controller 및 State Tree 기본 구조 구현, #35 적 체력·피격 및 사망 처리 구현

## 목적

공중형 근접 몬스터 한 종을 테스트 대상으로 구성하고, 조우·추적·행동 선택·회피·피격·사망까지의 전투 흐름을 구현한다.

행동의 상태 전환과 선택은 State Tree가 담당하고, 실제 행동 실행은 Gameplay Ability가 담당하도록 분리한다. 이를 통해 행동 로직을 몽타주·이동·Gameplay Effect·Gameplay Cue와 독립적으로 조합할 수 있도록 구성한다.

## 전투 의도 및 패턴

이 적은 지상에서 직선으로 접근하는 적이 아니라, 플레이어 주변을 부유하며 거리와 높이를 조절하는 근접형 일반 적이다. 공격만 반복하지 않고 거리별 공격, 위치 변경 대시, 총성 회피, 특수 행동을 섞어 전투 흐름을 만든다.

| 패턴 | 사용 거리·조건 | 역할 |
|---|---|---|
| 등장 (Emerge) | 스폰 포인트의 Emerge 등장이 실행될 때 (전투 섹터나 이벤트가 `Execute`) | 지면 아래 대기 상태에서 등장 연출을 재생한 뒤 전투를 시작한다. |
| 근접 할퀴기 (Melee) | Pattern Set의 거리 조건 | 타깃 근거리에서 팔 소켓 Sphere Trace로 피해를 적용한다. |
| 원거리 공격 (RangedAttack) | Pattern Set의 거리 조건 | 투사체를 발사해 거리를 유지한 상태에서도 압박한다. |
| 대시 (Dash) | Pattern Set의 거리 조건 | 스플라인을 따라 접근·이탈·측면 이동하며 전투 위치를 변경한다. |
| 빙의 (Possession) | Pattern Set의 거리 조건 | 주변 유효 대상에게 접근해 상태 효과와 Gameplay Cue를 적용한다. |
| 회피 (Evade) | 공격 Trace의 위협 확인, 쿨다운 충족, 인터럽트 가능 상태 | 공격 Trace에서 실제 위협 대상으로 확인되면 회피 요청 조건을 검사한다. |

## 주요 구조

```text
AI Perception / 스폰 포인트 (Emerge)
        ↓
AMonsterAIController
        ↓
ST_Enemy_Common (공용 State Tree)
        ↓ 패턴 선택 → 준비 → 실행 → 회복
Gameplay Ability
        ↓
Montage / Motion Warping / 이동 / 피해 처리
```

### State Tree와 Definition 구조

일반 몬스터의 전투는 공용 `ST_Enemy_Common`과 Enemy Definition 기반 데이터로 구성한다. 세부 State 이름, 바인딩 및 전이 우선순위는 에디터에서 확인한다.

```text
DA_Melee_Definition
├─ StatTableRowHandle / DisplayName / Emerge
├─ DA_Melee_AbilitySet
├─ DA_Melee_Pattern
└─ DA_Melee_MovementProfile
    ↓
Select Enemy Pattern → Prepare Enemy Pattern
    ↓
Execute Enemy Pattern → Wait Pattern Recovery → Enemy Idle Movement
```

인지·등장·피격·회피·타깃 상실은 공용 상태 흐름과 이벤트로 연결한다. 패턴은 거리·고도·체력·쿨다운 등의 조건으로 선택하며 실행 직전에 다시 검사한다. 전투 상태 흐름은 [#31](./Issue-031-AI-Controller-State-Tree.md)을 참고한다.

## 설계 결정

초기 State Tree 구조는 접근, 회전, 몽타주 재생을 개별 State Tree Task로 직접 조합하는 방식이었다. 근접 몬스터의 이동·대시·회피·피격·사망은 몽타주 구간, Motion Warping, Gameplay Effect, 쿨다운 등 Ability 단위의 처리가 필요하므로 다음과 같이 역할을 분리했다.

```text
State Tree
├─ 조우 / 추적 / 피격 / 회피 / 타깃 유실 상태 전환
├─ 행동 주기 관리
├─ 거리 조건과 가중치에 따른 행동 선택
└─ 선택한 Action Tag로 Ability 활성화

Gameplay Ability
├─ 몽타주 재생 및 섹션 제어
├─ Motion Warping 및 대시 이동
├─ Gameplay Effect / Gameplay Cue 적용
├─ 공격 판정 및 피해 적용
└─ 쿨다운, 종료 및 상태 복구
```

State Tree는 전투 흐름을, Gameplay Ability는 행동의 실제 실행을 담당한다.

## 구현 내용

### 조우 및 등장

- 등장 트리거는 스폰 포인트다. 전투 섹터(`StartEncounter`)나 이벤트가 Emerge 방식의 스폰 포인트를 `Execute`하면 `PlayEmerge()`가 AI를 깨우고 `RequestEncounter()`로 `Event.AI.EncounterRequested`를 전달한다. 초기 구현은 몬스터의 Encounter Sphere 오버랩이 이 요청을 냈지만, 스폰 포인트 기반으로 바뀌면서 그 경로는 쓰이지 않는다.
- `Dormant`에서 등장 행동을 선택하고 `GA_Emerge`를 실행한다.
- 등장 몽타주 완료 후 `Combat`으로 전환한다.
- 등장 전에는 전투 행동을 수행하지 않는다.
- 전투 중 타깃을 놓쳐도 스폰 포인트로 돌아가지 않고 놓친 자리에서 대기한다. 벽·천장에서 나온 몹이 그 자리로 되돌아가는 것은 어색하고 그 위치에는 네비메시가 없을 수도 있어서, Emerge 등장 방식은 순찰 영역을 제공하지 않는다. StateTree의 Idle은 순찰 영역이 없는 몹을 제자리 대기로 보낸다.

### 이동과 패턴 구성

일반 추격은 AI Controller와 이동 컴포넌트가 처리하고, 패턴 준비 이동은 `IEnemyPatternMovement` 경로를 사용한다. `State.AI.ActionCommitted`는 일반 추격과 GA 이동이 겹치지 않도록 제한한다.

- Definition의 Movement Profile로 이동 방식, 속도·가속·도착 판정 및 높이 설정을 적용한다.
- Pattern Set에서 실행 조건, 가중치, 이동 후보와 Recovery 범위를 설정한다.
- `bMoveDuringAbility`를 사용하는 후보는 준비 완료를 기다리지 않고 GA와 이동을 병행한다.
- `RepositionForAction`은 특성 조건과 위치 보정 프로필을 확인해 재배치한다.
- GA 실행 중 상태 이탈 처리와 회피 허용은 패턴의 `InterruptPolicy`를 따른다.
- 타깃 기억, 우회 이동과 높이 보정은 컨트롤러 및 이동 컴포넌트 설정으로 관리한다.

### 공격 실행

서버가 Definition의 Ability Set에 등록된 GA를 지급한다. 근접 공격은 몽타주의 HitWindow, 원거리 공격은 발사 이벤트와 투사체로 판정한다. 대시·빙의 등 특수 행동은 해당 GA가 이동과 효과의 시작·종료를 관리한다. 거리·피해·쿨다운은 Pattern Set, 공격 설정과 몽타주에서 조정한다.

### 피격 및 사망 연동

- 누적 피해·피격 횟수 등 피격 반응 조건을 확인한 뒤 일반 `GA_HitReact`를 실행한다.
- 피격 중에는 `State.Groggy` 상태를 Evaluator에 전달하고 `HitReactWait`에서 행동 선택을 멈춘다.
- 체력이 0 이하가 되면 `State.Dead`, `Event.Death`, `GA_Death` 흐름으로 사망을 처리한다.
- 사망 몽타주는 공격자 방향에 따라 Front / Back / Left / Right 섹션을 선택한다.
- `Pose Snapshot`으로 사망 몽타주 종료 뒤에도 사망 자세를 유지한다.

### 시각·사운드 처리

- 빙의 행동은 Gameplay Effect의 `State.AI.Possessed`와 Gameplay Cue를 통해 상태 및 Niagara 효과를 적용한다.
- 근접 공격 몽타주에는 타격 시점과 맞춘 Sound Notify를 배치한다.

### Definition과 특성 데이터

`DA_Melee_Definition`은 표시 이름, 스탯 행, Ability Set, Pattern Set, Movement Profile과 기본 등장 몽타주를 참조한다. 서버는 Ability Set에서 GA를 지급하고 기본 Pattern Set을 활성화한다. 변형 특성은 별도 Trait Pool에서 선택하며, 선택된 GE 및 Overlay 정보를 적용한다. 시작 방식·순찰·타깃 상실 대응은 캐릭터와 스폰 배치 설정에 둔다.

## 관련 구현

### C++

- `Source/TeamProject_Fin/GameCore/AI/MonsterAIController.h/.cpp`
- `Source/TeamProject_Fin/FSTT_SelectEnemyPattern.h/.cpp`
- `Source/TeamProject_Fin/FSTT_PrepareEnemyPattern.h/.cpp`
- `Source/TeamProject_Fin/FSTT_ExecuteEnemyPattern.h/.cpp`
- `Source/TeamProject_Fin/FSTT_WaitPatternRecovery.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Enemy/Data/EnemyDataAssets.h`
- `Source/TeamProject_Fin/FSTT_ActivateAbilityByTag.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Movement/FlyingMonsterMovementComponent.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_Emerge.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_Dash.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_Evade.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_MeleeAttack.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_RangedAttack.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_Possession.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Animation/ANS_MeleeAttackHit.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/DamageSystemLibrary.h/.cpp`

### Blueprint / Asset

- `Content/Monster/DA/Melee/DA_Melee_Definition`
- `Content/Monster/DA/Melee/DA_Melee_AbilitySet`
- `Content/Monster/DA/Melee/DA_Melee_Pattern`
- `Content/Monster/DA/Melee/DA_Melee_MovementProfile`
- `Content/Monster/AI/ST/ST_Enemy_Common`
- `Content/Monster/AI/Controller/BP_MonsterAIController_Melee`
- `Content/Monster/BP/BP_Monster_Melee`
- `Content/Monster/GAS/GA/BP_GA_Emerge`
- `Content/Monster/GAS/GA/BP_GA_Dash`
- `Content/Monster/GAS/GA/BP_GA_Evade_Melee`
- `Content/Monster/GAS/GA/BP_GA_Melee_RangedAttack_Normal`
- `Content/Monster/GAS/GA/BP_GA_Possession`
- `Content/Monster/Anim/Montage/Melee/AM_Melee_Claw`
- `Content/Monster/Anim/Montage/Melee/AM_Melee_Emerge_Floor`
- `Content/Monster/Anim/Montage/Melee/AM_Melee_Evade`

## 사용 방법

1. 몬스터 Blueprint에 `AMonsterAIController` 기반 AI Controller를 지정하고, 그 컨트롤러가 `ST_Enemy_Common`을 실행하게 한다.
2. State Tree에 `Sentinel Perception`, `Monster Combat State` Evaluator를 추가하고 필요한 출력값을 바인딩한다.
3. 스폰 포인트를 섹터 조각 BP(ChildActor)나 레벨에 배치하고 등장 방식을 Emerge로 지정한다. 전투 섹터나 이벤트가 스폰 포인트를 `Execute`하면 `PlayEmerge()`가 `RequestEncounter()`를 호출해 `Event.AI.EncounterRequested`를 전송한다.
4. `GA_Emerge`, 대시, 근접 공격, 원거리 공격, 회피, 빙의 Ability를 Definition이 참조하는 Ability Set에 등록하고 실행 태그를 맞춘다.
5. 몬스터의 패턴 세트 데이터 에셋에서 패턴별 Ability Tag, 거리 범위, 가중치, 쿨다운, 회복 시간(`RecoveryDuration`)을 전투 의도에 맞게 설정한다.
6. 공격 몽타주에 Motion Warping, 공격 판정 Notify State, Sound Notify를 배치하고 `GE_Damage_Base`를 지정한다.
7. Player Capsule과 Monster Projectile Collision Component의 `EnemyAttack` 충돌 반응을 Block으로 설정한다.

## 테스트 및 확인 항목

개별 거리·확률·피해량·쿨다운 수치는 DA, BP 및 몽타주 설정에서 관리한다. 이 문서의 확인 항목은 실행 검증 체크리스트이며 완료 결과를 의미하지 않는다. Blueprint 내부 참조와 ST의 실제 전이는 에디터에서 확인한다.

- [ ] 등장 트리거 시 등장 행동이 한 번 실행되는지 확인 (초기에는 Encounter Sphere 진입, 현재는 스폰 포인트 `Execute`)
- [ ] 등장 종료 전에는 전투 행동과 이동이 시작되지 않는지 확인
- [ ] 타깃 탐지 후 부유 이동으로 추적하는지 확인
- [ ] 거리별 후보 행동만 선택되고 Weight 기준으로 선택되는지 확인
- [ ] 대시, 근접 공격, 원거리 공격, 빙의 행동 실행 확인
- [ ] 공격 Trace 위협 확인 시 인터럽트 정책에 따라 회피하는지 확인
- [ ] 타깃 유실 시 `LostTarget` 대기, 재탐지 시 `Combat` 복귀 확인
- [ ] 근접 스윕 및 원거리 투사체가 플레이어에게 피해를 적용하는지 확인
- [ ] 피격 중 행동 선택이 중단되고 피격 몽타주가 중복 재생되지 않는지 확인
- [ ] 사망 방향별 몽타주, Pose Snapshot, AI 중지 및 액터 제거 확인
- [ ] 근접 공격 Sound Notify 재생 확인

## 후속 리팩토링 변경점

Issue #32의 최초 구현 이후, Phantom을 포함한 공통 몬스터 기반의 재사용성과 전투 흐름을 보완하기 위해 다음과 같이 변경했다.

### Emerge

- 등장 방식을 `Floor`, `Wall`, `Ceiling`으로 확장했다.
- 등장 타입과 등장 몽타주 오버라이드를 레벨 인스턴스별로 지정할 수 있도록 변경했다.
- `Wall`과 `Ceiling` 등장 시 `Turn_Left` 또는 `Turn_Right` 섹션을 선택할 수 있도록 추가했다.
- 타깃 방향과 몬스터의 현재 자세를 기준으로 좌우 섹션을 계산하도록 변경했다.
- 선택적으로 Motion Warping을 적용해 Turn 구간 동안 타깃 방향과 월드 Up 방향을 보정하도록 했다.
- Encounter 이벤트를 통해 `Dormant` 상태에서 Emerge를 시작하고, 등장 몽타주가 끝난 뒤 `Combat`으로 진입하도록 정리했다.

### Evade

- 총성 청취만으로 회피하지 않고, 공격 Trace에서 실제 위협 대상이 확인된 경우에만 회피 요청을 보내도록 변경했다.
- `MonsterAIController`에서 인터럽트 가능 여부, 타깃 보유 여부, 회피 Ability 존재 여부, 쿨다운을 검사한다.
- 조건을 통과하면 `Event.AI.Evade.Requested`를 State Tree에 전달하고, State Tree의 Evade 상태에서 `GA_Evade`를 실행한다.
- 회피 중에는 무적 태그를 적용하고, 회피 Ability 종료 시 무적 태그를 제거하도록 했다.
- 회피 성공 여부에 따라 실제 피해 적용을 생략하도록 공격 판정 흐름을 보완했다.

### 공통 기반 보완

- State Tree는 상태 전환과 행동 선택을 담당하고, Gameplay Ability는 몽타주·판정·무적·종료 처리를 담당하도록 역할을 정리했다.
- 몬스터 공통 Evaluator에 `HasEncountered`, `OwnedTags` 등의 상태 정보를 연결했다.
- 근접·원거리 공격 사운드를 몽타주 Notify에서 재생할 수 있도록 관련 에셋을 추가했다.

## 후속 작업

- Gameplay Cue 연동 정리 및 시각 효과 조정
- 행동별 거리·가중치·쿨다운 밸런스 조정
- 체력바 및 전투 상태 UI 갱신 검증
- [#36 보상 드랍](./Issue-036-Enemy-Drop-Reward.md) 연동 검증
- 실제 전투 시나리오 및 멀티플레이 환경 검증
