# #34 엘리트 일반 적 구현

## Issue

- Issue 번호: #34
- 기능명: 엘리트 일반 적 구현
- 담당자: horimihorimi
- 담당 영역: Enemy AI / GAS / Animation / Combat
- 관련 Sprint: Sprint 3
- 선행 이슈: #31 AI Controller 및 State Tree 기본 구조 구현, #32 근접형 일반 적 구현, #33 원거리형 일반 적 구현

## 목적

근거리 회전 공격·레이저·드론 소환을 조합하는 엘리트 몬스터의 전투 흐름을 구현한다.

State Tree가 공격 패턴과 이동 후보를 선택한다. 재배치 후보가 선택되면 `MonsterCombatPositioningComponent`가 거리와 방향을 평가하고 프로필 정책에 따라 은신 순간이동을 수행한다. 재배치는 Teleport 특성 조건을 충족한 경우에 사용한다.

## 전투 의도 및 패턴

엘리트 몬스터는 일반 몬스터보다 넓은 공간을 사용하며, Teleport 특성이 적용되면 공격 전 순간이동으로 플레이어의 대응 방향을 바꿀 수 있는 적이다. 근거리에서는 전진형 회전 공격으로 압박하고, 거리가 벌어지면 레이저와 드론으로 이동을 제한한다.

| 패턴 | 역할 |
|---|---|
| 등장 (Emerge) | 인카운터 시작 시 등장 몽타주를 재생한 뒤 전투를 시작한다. |
| 회전 공격 (Spin) | 전진하면서 회전해 근거리의 플레이어에게 다단 히트 피해를 준다. |
| 구르기 회전 공격 (RollSpin) | 구르며 전진한 뒤 회전 공격으로 이어간다. |
| 레이저 공격 (Beam) | 일정 거리에서 제한된 회전 속도로 타깃을 추적하는 지속형 레이저를 발사한다. |
| 드론 소환 (Drones) | 폭발형 드론을 배치한다. 드론은 일정 시간 뒤 전기와 폭발 피해를 준다. |
| 순간이동 재배치 (Teleport Reposition) | Teleport 특성을 받은 엘리트만, 선택한 공격의 거리·방향 조건에 맞는 위치로 은신 순간이동한다. |

## 주요 구조

```text
AI Perception / 스폰 포인트 (Emerge)
        ↓
AMonsterAIController
        ↓
ST_Enemy_Common (공용 State Tree)
        ↓ Select Enemy Pattern
현재 패턴 (DA_Elite_Pattern) + 이동 후보
        ↓
Prepare Enemy Pattern
        ├─ 이동 후보 없음 → 제자리에서 바로 시전
        ├─ 일반 이동 후보 → 지상 이동 컴포넌트가 이동
        └─ RepositionForAction (Teleport 특성 필요) → MonsterCombatPositioningComponent → GA_TeleportReposition
        ↓
Execute Enemy Pattern (공격 GA 실행)
        ↓
Wait Pattern Recovery → Enemy Idle Movement → 다음 패턴
```

### State Tree 구조

엘리트도 근접 몬스터와 같은 공용 `ST_Enemy_Common`을 쓴다(`BP_MonsterAIController_Elite`가 지정). 전체 구조는 [Issue-32 근접 몬스터](Issue-32-Melee-Enemy)의 State Tree 구조를 따르고, 엘리트의 전투 흐름은 그중 `CombatLoop`다. 엘리트 전용 State Tree(`ST_Enemy_Elite`)는 폐기했고, 엘리트만의 차이는 패턴 세트(`DA_Elite_Pattern`), 지상 이동 컴포넌트, 특성으로 만든다.

```text
Combat [HasEncountered == true, HasTarget == true]
└─ CombatLoop
   ├─ SelectPattern (Select Enemy Pattern)     성공 → PreparePattern, 실패 → IdleMovement
   ├─ PreparePattern (Prepare Enemy Pattern)   성공 → ExecutePattern, 실패 → IdleMovement
   ├─ ExecutePattern (Execute Enemy Pattern)   성공 → Recovery, 실패 → IdleMovement
   ├─ Recovery (Wait Pattern Recovery)         성공·실패 → IdleMovement
   └─ IdleMovement (Enemy Idle Movement)       성공·실패 → SelectPattern
```

`Execute Enemy Pattern`이 시작될 때 다음 패턴을 미리 골라 두기 때문에, 회복 대기가 끝나면 지연 없이 다음 패턴이 이어진다.

## 설계 결정

엘리트 공격은 행동 선택, 위치 선정, 순간이동 연출, 실제 공격 실행을 분리했다.

```text
State Tree
├─ 전투 상태 전환
├─ 공격 패턴 선택 (Select Enemy Pattern)
├─ 이동·재배치 준비 (Prepare Enemy Pattern)
└─ 선택한 패턴의 GA 실행 (Execute Enemy Pattern)

MonsterCombatPositioningComponent
├─ 공격별 위치 프로필 조회
├─ 현재 위치 유효성 검사
├─ 후보 위치 생성 및 검증
└─ 후보 점수 계산

GA_TeleportReposition
├─ 이동 및 인터럽트 잠금
├─ Cloak / Reveal 연출
├─ Notify 시점 좌표 변경
└─ 종료 시 상태 복구

공격 Gameplay Ability
├─ 몽타주 재생
├─ 공격 판정
├─ 피해 적용
└─ 종료 처리
```

위치 준비 중에도 선택된 패턴은 원래 공격을 유지한다. 순간이동은 공격을 대체하는 행동이 아니라 선택된 공격을 실행하기 위한 선행 행동이다.

## 구현 내용

### 이동 방식

기존 근접·원거리 몬스터는 타깃을 인지하면 `MonsterMovementComponent`를 통해 지속적으로 타깃을 추적한다.

- 지상형 몬스터는 NavMesh 경로를 따라 `MoveToActor()`로 접근한다.
- 비행형 몬스터는 타깃과의 거리에 따라 접근 이동과 좌우 Strafe를 전환한다. 타깃까지 직선이 벽에 막혀 있으면 Strafe를 끄고, 위치 이동과 같은 바닥 네비 경로를 따라 벽을 돌아 접근한다. 경로를 구하지 못하면 직선으로 접근한다. 패턴이 정한 이동(접근·앵커 이동)에는 적용하지 않는다. 경로를 따라갈 때는 경로 지점의 바닥과 천장으로 구한 몸 통과 범위(캡슐 기준) 안으로 부유 높이를 미리 맞추고, 몸이 지나갈 수 없는 경로는 따라가지 않는다. 자세한 내용은 Issue-32의 "경로 통과성 검사와 경로 기반 높이"를 따른다.
- 시선이 끊겨도 `SightMemoryDuration` 설정 동안은 타깃이 유지되므로, 벽 뒤로 돌아간 플레이어를 위 경로 이동으로 계속 쫓는다.
- 공격 GA가 실행되는 동안에만 일반 추적 이동을 잠그고, 공격이 끝나면 다시 추적한다.

엘리트도 일반 지상 몬스터와 같은 기본 이동을 쓴다. `BP_Monster_Elite`는 `GroundMonsterMovementComponent`를 쓰고, 엘리트 AIController의 `bEnablePursuitMovement`는 BP에서 덮어쓰지 않아 기본값(켜짐)이다. 지속 추적 이동을 끄고 순간이동만 쓰던 초기 설계는 폐기했다. 공격 GA가 `State.AI.ActionCommitted`를 들고 있는 동안에는 이동하지 않고, 그 태그가 없는 구간(공격 사이 회복 대기 등)에는 컨트롤러 추적이 `MoveToActor`로 네비메시 경로를 따라 이동한다.

순간이동은 기본 이동이 아니라 **Teleport 특성(`Enemy.Trait.Teleport`)을 받은 엘리트에게만** 패턴 사전 이동에 섞인다. 패턴의 이동 후보 `RepositionForAction`은 이 태그가 있을 때만 뽑히고(`ResolveCurrentMovementOption`), 태그가 없으면 다른 이동 후보만 뽑는다. 뽑을 후보가 없으면 제자리에서 바로 시전한다.

```text
타깃 인지
→ 공격 패턴 선택
→ Teleport 특성이 있고 재배치가 필요하면 은신 순간이동 → 공격 실행
→ 특성이 없거나 현재 위치가 적합하면 그 자리에서 공격 실행
→ 공격 종료 후 회복 대기 동안 기본 이동(네비메시 추적)으로 접근
→ 다음 패턴
```

따라서 엘리트의 위치 변화는 다음 세 경로에서 발생한다.

1. Teleport 특성이 있을 때 공격 전 `GA_TeleportReposition`이 수행하는 순간이동
2. `Spin`, `RollSpin` 공격 몽타주가 수행하는 전진 이동
3. 공격 사이에 도는 기본 이동(지상 몹의 네비메시 추적)

순간이동 목적지는 선택된 공격의 거리 조건을 기준으로 타깃 주변 NavMesh에서 찾는다. 공격 사이에 실제로 걸어서 이동하는 시간은 패턴의 `RecoveryDuration`이 정한다.

### 행동 선택

엘리트의 패턴은 `DA_Elite_Pattern` 패턴 세트에 정의되고, 각 패턴은 패턴 ID와 실행할 Ability 태그를 가진다.

| 패턴 ID | Ability 태그 | 패턴 |
|---|---|---|
| `Pattern.Enemy.Elite.Spin` | `Ability.Enemy.Elite.Spin` | 전진 회전 공격 |
| `Pattern.Enemy.Elite.RollSpin` | `Ability.Enemy.Elite.RollSpin` | 구르기 후 전진 회전 공격 |
| `Pattern.Enemy.Elite.Beam` | `Ability.Enemy.Elite.Beam` | 추적 레이저 공격 |
| `Pattern.Enemy.Elite.Drones` | `Ability.Enemy.Elite.Drones` | 폭발형 드론 소환 |

이전 구현에서 쓰던 `Action.AI.Elite.*` 태그는 옛 State Tree가 아직 참조할 수 있어 정의만 남아 있다.

`Select Enemy Pattern`(`UEnemyPatternComponent::SelectPattern`)은 체력 비율, 쿨다운, 연속 사용 제한, 고도 조건, 타깃과의 거리 범위(`MinDistance`~`MaxDistance`), 공격 GA 발동 가능 여부를 통과한 패턴만 후보로 구성하고 가중치로 하나를 고른다. 현재 패턴이 실행되는 동안 다음 패턴을 미리 골라 두고, 실행 직전에 조건을 다시 확인한다. 선택된 패턴은 `EnemyPatternComponent`가 들고 있으며 State Tree의 Prepare, Execute, Recovery 태스크가 그 값을 읽는다.

### 공격별 위치 준비

패턴의 이동 후보가 `RepositionForAction`이면 `FSTT_PrepareEnemyPattern`이 재배치를 `MonsterCombatPositioningComponent`에 넘기고, 이 컴포넌트가 패턴의 Ability 태그에 대응하는 `FMonsterActionPositionProfile`을 조회한다. 이 이동 후보는 Teleport 특성(`Enemy.Trait.Teleport`)이 있는 몬스터에게만 뽑힌다.

프로필의 주요 설정은 다음과 같다.

| 속성 | 설명 |
|---|---|
| `DesiredMinDistance` | 타깃으로부터 떨어질 최소 거리 |
| `DesiredMaxDistance` | 타깃으로부터 떨어질 최대 거리 |
| `bRequireLineOfSight` | 공격 전 타깃 시야 확보 필요 여부 |
| `RepositionPolicy` | 재배치를 실행할 조건 |
| `RepositionAbilityTag` | 재배치에 사용할 GA 태그 |
| `DirectionPreference` | 타깃 정면·측면·후면 중 선호 방향 |
| `DirectionSampleCount` | 타깃 주변을 나눌 방향 후보 수 |
| `DistanceSampleCount` | 최소·최대 거리 사이의 단계 수 |

재배치 정책은 세 가지다.

| 정책 | 동작 |
|---|---|
| `Never` | 순간이동하지 않는다. 현재 위치가 부적합하면 공격 준비에 실패한다. |
| `WhenOutOfRange` | 현재 위치가 공격 조건에 맞지 않을 때만 순간이동한다. |
| `Always` | 현재 위치가 적합해도 공격 전에 항상 새로운 위치를 찾는다. |

### 순간이동 후보 선정

후보는 타깃을 중심으로 `DesiredMinDistance`와 `DesiredMaxDistance` 사이에 생성한다.

```text
후보 생성
→ NavMesh 투영
→ 캡슐 배치 가능 여부 검사
→ 바닥 경사 검사
→ 현재 위치와 높이 차 검사
→ 필요 시 타깃 시야 검사
→ 유효 후보 점수 계산
→ 최고 점수 후보 선택
```

후보 점수에는 거리 적합도, 방향 선호도, 시야, 주변 여유 공간을 가산하고 최근 순간이동 위치와 지나치게 짧은 이동 거리를 감점한다. 연출상 선호 조건은 후보를 제거하지 않고 점수로 처리해, 설정 조합 때문에 모든 후보가 사라지는 상황을 줄였다.

### 은신 순간이동

`GA_TeleportReposition`은 `Ability.Enemy.Elite.TeleportReposition` 태그로 실행된다(이전 이름 `Action.AI.TeleportReposition`은 태그 리다이렉트로 이 태그를 가리킨다).

```text
GA 활성화
→ AI 이동 및 인터럽트 잠금
→ Cloak 시작
→ 순간이동 몽타주 재생
→ Event.AI.Teleport.Commit 수신
→ 준비된 좌표로 이동
→ Reveal 시작
→ 몽타주 종료
→ 이동 및 인터럽트 잠금 해제
```

실제 좌표 변경은 몽타주 시작 시점이 아니라 `Event.AI.Teleport.Commit` Notify 시점에 수행한다. Commit 전에 GA가 취소되면 준비된 위치를 폐기하고 Cloak 값을 즉시 복구한다.

`MonsterCloakComponent`는 평상시 원본 머티리얼을 유지하고, 은신 구간에만 동적 머티리얼을 적용한다. Reveal이 끝나면 원본 머티리얼로 되돌린다.

### 근거리 회전 공격

- `Spin`은 전진하면서 회전해 다단 히트 피해를 적용한다.
- `RollSpin`은 구르며 전진한 뒤 회전 공격으로 이어진다.
- 두 패턴 모두 몽타주에 배치된 공격 판정 Notify State가 실제 타격 구간을 결정한다.
- 이동 중 반복 판정이 발생하므로 동일 대상의 재피격 횟수 또는 간격을 제한한다.

### 레이저 공격

`ANS_BeamHit`은 지정 소켓에서 `BeamLength`만큼 트레이스를 수행하며, 판정 결과와 Niagara 빔의 끝점을 함께 갱신한다.

- `OwnerForward`: 몬스터 전방을 기준으로 발사한다.
- `SocketForward`: 지정 소켓의 전방을 기준으로 발사한다.
- `Target`: 타깃 방향을 추적한다.

`Target` 모드의 회전 속도는 `TurnSpeed`로 제한한다. 벽에 막히면 빔 연출과 공격 판정이 충돌 지점에서 함께 끝난다. 지속 공격이므로 `HitCount` 또는 `ReHitInterval`을 설정해 다단 히트 횟수를 제한한다.

### 드론 소환

드론은 개별 AI가 아니라 `ADroneSwarm` 액터 하나가 무리 전체를 관리한다.

```text
Deploy → Hover → Settle → Discharge → Finished
```

| 단계 | 동작 |
|---|---|
| `Deploy` | 엘리트의 사출 소켓에서 배치 지점까지 곡선으로 이동한다. |
| `Hover` | 배치 지점에서 대기하며 플레이어에게 회피 시간을 제공한다. |
| `Settle` | 움직임을 멈추고 최종 공격 위치를 확정한다. |
| `Discharge` | 드론 사이 전기와 폭발 피해를 적용한다. |
| `Finished` | 연출 종료 후 드론 무리를 제거한다. |

드론 배치는 사출 시점에 확정되며 이후 타깃을 계속 추적하지 않는다. 엘리트 본체가 먼저 사망해도 이미 사출된 드론이 공격을 완료할 수 있도록 `ADroneSwarm`이 자체 ASC와 사출 당시 공격력 정보를 가진다.

### 등장 무적

인카운터 시작 시 프레젠테이션과 충돌이 활성화된 뒤 State Tree가 `GA_Emerge`를 실행할 때까지 짧은 피격 공백이 생길 수 있다. 이를 막기 위해 프레젠테이션 공개 전에 임시 `State.Invincible`을 적용하고, `GA_Emerge` 종료 시 임시 무적과 GA 소유 무적을 함께 해제한다.

### Definition과 특성 데이터

`DA_Elite_Definition`은 표시 이름, 스탯 행, Ability Set, Pattern Set, Movement Profile과 기본 등장 몽타주를 참조한다. 서버는 Ability Set에서 GA를 지급하고 기본 Pattern Set을 활성화한다. 변형 특성은 별도 Trait Pool에서 선택하며, 선택된 GE 및 Overlay 정보를 적용한다. 시작 방식·순찰·타깃 상실 대응은 캐릭터와 스폰 배치 설정에 둔다.

## 관련 구현

### C++

- `Source/TeamProject_Fin/GameCore/Combat/MonsterCombatPositioningComponent.h/.cpp`
- `Source/TeamProject_Fin/FSTT_PrepareEnemyPattern.h/.cpp` (재배치 이동 후보를 `MonsterCombatPositioningComponent`에 넘김)
- `Source/TeamProject_Fin/GameCore/Ability/GA_TeleportReposition.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Combat/MonsterCloakComponent.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Animation/ANS_BeamHit.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Animation/AN_SpawnDroneSwarm.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Combat/DroneSwarm.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Tags/GameplayTagSystem.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_Emerge.h/.cpp`

### Blueprint / Asset

- `Content/Monster/DA/Elite/DA_Elite_Definition`
- `Content/Monster/DA/Elite/DA_Elite_AbilitySet`
- `Content/Monster/DA/Elite/DA_Elite_Pattern`
- `Content/Monster/DA/Elite/DA_Elite_MovementProfile`
- `Content/Monster/BP/BP_Monster_Elite`
- `Content/Monster/AI/Controller/BP_MonsterAIController_Elite`
- `Content/Monster/AI/ST/ST_Enemy_Common`
- `Content/Monster/Anim/ABP_Elite`
- `Content/Monster/Anim/BS/BS_Elite_Move`
- `Content/Monster/GAS/GA/BP_GA_TeleportReposition`
- `Content/Monster/GAS/GA/BP_GA_Elite_SpinAtk`
- `Content/Monster/GAS/GA/BP_GA_Elite_RollSpinAtk`
- `Content/Monster/GAS/GA/BP_GA_Elite_BeamAtk`
- `Content/Monster/GAS/GA/BP_GA_Elite_SpawnDrone`
- `Content/Monster/Actor/BP_DroneSwarm`
- `Content/Monster/Anim/Montage/Elite/AM_Elite_Teleport`
- `Content/Monster/Anim/Montage/Elite/AM_Elite_SpinAtk`
- `Content/Monster/Anim/Montage/Elite/AM_Elite_RollSpinAtk`
- `Content/Monster/Anim/Montage/Elite/AM_Elite_TurretAtk`
- `Content/Monster/Anim/Montage/Elite/AM_Elite_SpawnDrones`
- `Content/Monster/DT/DT_MonsterAttackDamage`

## 사용 방법

1. 엘리트 Blueprint에 `BP_MonsterAIController_Elite`를 AI Controller로 지정한다.
2. AIController가 공용 `ST_Enemy_Common`을 실행하게 한다.
3. 엘리트 Blueprint에 `MonsterCombatPositioningComponent`와 `MonsterCloakComponent`를 추가한다.
4. `MonsterCombatPositioningComponent.ActionProfiles`에 네 공격 태그의 거리·방향·재배치 정책을 등록한다.
5. 재배치가 필요한 프로필의 `RepositionAbilityTag`를 `Ability.Enemy.Elite.TeleportReposition`으로 지정한다.
6. `DA_Elite_Definition`이 참조하는 `DA_Elite_AbilitySet`에 순간이동 및 공격 GA를 등록한다.
7. `DA_Elite_Pattern`의 패턴마다 이동 후보에 `RepositionForAction`을 넣는다. 순간이동은 Teleport 특성을 받은 엘리트에게만 뽑히고, 특성이 없으면 다른 이동 후보를 쓰거나 제자리에서 시전한다. 별도의 State Tree 작업은 필요 없고 `ST_Enemy_Common`의 `Prepare Enemy Pattern`이 재배치를 처리한다.
8. 순간이동 몽타주에 `Event.AI.Teleport.Commit` 이벤트를 배치한다.
9. 공격 몽타주에 각 공격의 판정 Notify State와 사운드·연출 Notify를 배치한다.
10. 드론 몽타주에 `Spawn Drone Swarm` Notify를 배치하고 사출 소켓을 지정한다.
