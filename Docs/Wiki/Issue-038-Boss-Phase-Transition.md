# #38 보스 페이즈 전환 시스템 구현

## Issue

- Issue 번호: #38
- 기능명: 보스 페이즈 전환 시스템 구현
- 담당자: horimihorimi
- 담당 영역: Boss Combat / State Tree / GAS / Animation
- 관련 Sprint: Sprint 5
- 선행 이슈: #37 보스 기본 캐릭터 구현

## 목적

보스 체력 구간에 따라 페이즈를 순차적으로 전환하고, 전환 연출이 완료된 뒤 해당 페이즈의 Pattern Set을 활성화하는 시스템을 구현한다.

페이즈 판정, 전환 연출, 패턴 교체를 분리해 체력 변화 중 패턴이 중복 실행되거나 전환 도중 다음 페이즈가 중복 요청되는 문제를 방지한다.

## 주요 구조

```text
Health 변경
    ↓
UBossPhaseComponent
    ↓
Phase Transition Requested
    ↓
State Tree
├─ BeginBossPhaseTransition
├─ ExecuteBossPhaseTransition
└─ CompleteBossPhaseTransition
    ↓
새 Phase Pattern Set 활성화
```

```text
BP_Boss_PhaseSet
├─ Phase 1 → DA_Boss_Phase1
├─ Phase 2 → DA_Boss_Phase2
└─ Phase 3 → DA_Boss_Phase3
```

## 페이즈 데이터

각 페이즈는 `FEnemyBossPhaseDefinition`으로 구성한다.

| 필드 | 설명 |
|---|---|
| `PhaseId` | 페이즈 식별 태그 |
| `HealthThreshold` | 해당 페이즈로 진입하는 체력 비율 |
| `TransitionAbilityTag` | 전환 시 실행할 GA 태그 |
| `PatternSet` | 전환 완료 후 활성화할 패턴 세트 |
| `MeshScaleMultiplier` | 페이즈 유지 중 메시 스케일 |
| `CapsuleRadiusMultiplier` | 페이즈 유지 중 캡슐 반경 배율 |
| `CapsuleHalfHeightMultiplier` | 페이즈 유지 중 캡슐 반높이 배율 |
| `MovementProfileOverride` | 페이즈 변경 시 적용할 이동 프로필 |
| `bStationary` | 일반 공전과 패턴 이동을 막고 제자리 전투 사용 |
| `FacingMode` | 패턴 이동 중 타깃 또는 아레나 중심을 바라보는 기준 |
| `bLockCombatDuringTransition` | 전환 중 전투 잠금 설정 |

배열은 반드시 P1 → P2 → P3 순서로 구성한다.

| 인덱스 | 페이즈 | 체력 조건 | 전환 GA | 패턴 |
|---|---|---|---|---|
| 첫 항목 | P1 | 시작 페이즈 | 없음 | `DA_Boss_Phase1` |
| 다음 항목 | P2 | 기획 체력값 이하 | `Ability.Enemy.Boss.Phase2Transition` | `DA_Boss_Phase2` |
| 다음 항목 | P3 | 기획 체력값 이하 | `Ability.Enemy.Boss.Phase3Transition` | `DA_Boss_Phase3` |

## 설계 결정

### 페이즈 확정 시점

체력 임계값을 넘는 즉시 현재 페이즈를 변경하지 않는다.

```text
체력 임계값 도달
→ 전환 요청 상태
→ 전환 GA 재생
→ 전환 GA 종료
→ 페이즈 인덱스 확정
→ 새 Pattern Set 활성화
```

전환 중에는 이전 패턴을 정리하고 새 패턴 실행을 막는다. 따라서 전환 몽타주 중 P2 또는 P3 공격이 조기에 실행되지 않는다.

### 중복 전환 방지

`UBossPhaseComponent`의 `bTransitionInProgress`를 사용한다.

- 전환 요청 직후 `true`로 설정한다.
- 전환이 끝난 뒤 `CompletePhaseTransition()`에서 `false`로 복구한다.
- 전환이 진행 중이면 체력이 추가로 감소해도 새 전환을 요청하지 않는다.

## 구현 내용

### 체력 비율 감시

- `ABossMonsterCharacter`가 Health Attribute 변경 Delegate를 등록한다.
- 현재 체력 / 최대 체력으로 체력 비율을 계산한다.
- `UBossPhaseComponent::TryRequestPhaseTransition()`에 전달한다.
- 다음 페이즈의 `HealthThreshold` 이하일 때 전환을 요청한다.

### State Tree 전환 흐름

#### `Boss Phase Transition In Progress`

- `UBossPhaseComponent::IsPhaseTransitionInProgress()`를 확인한다.
- 전환 요청 상태일 때 전환 Branch로 진입한다.

#### `Begin Boss Phase Transition`

- 실행 중인 Pattern Ability를 취소한다.
- `EnemyPatternComponent`의 런타임 상태를 초기화한다.
- 보스 일반 이동 준비 상태를 취소한다.
- AI를 인터럽트 가능한 상태로 전환한다.

#### `Execute Boss Phase Transition`

- Pending Phase의 `TransitionAbilityTag`를 읽는다.
- ASC에서 해당 GA를 실행한다.
- 활성 Ability Spec Handle을 보관하고 GA가 끝날 때까지 대기한다.

#### `Complete Boss Phase Transition`

- `CompletePhaseTransition()`으로 현재 페이즈 인덱스를 확정한다.
- 새 페이즈의 Pattern Set을 `EnemyPatternComponent`에 적용한다.
- 보스 Character가 새 Phase Form을 적용한다.

### 전환 GA

#### 공통 전환

`UGA_BossPhaseTransitionBase`는 공통 전환 흐름을 처리한다.

- 전환 몽타주 재생
- `Reposition` 노티파이 이벤트 수신 후 앵커 기준 위치 재배치
- `Rise` 노티파이 이벤트 수신 후 목표 앵커 위치까지 이동
- 이동 완료 시 다음 몽타주 섹션 연결
- 전환 중 `State.AI.ActionCommitted`, `State.Invincible` 부여
- Reposition 및 Rise 이벤트 중복 호출 방어

#### P2 전환

`UGA_BossPhaseTransition`은 링 Spawn 노티파이에서 링 버스트를 생성하고, 링 Launch 노티파이에서 대기 중인 링을 발사한다. 모든 링 버스트가 끝나면 다음 몽타주 섹션으로 진행한다.

#### P3 전환

`BP_GA_BossPhase3Transition`은 P3 전환 몽타주를 재생하고 재배치와 상승 이동을 수행한다. `AN_BossFormScale` Notify State가 커브 값을 매 프레임 적용하며, 전환이 끝나면 P3의 영구 메시 및 캡슐 배율을 적용한다.

### 전환 중 다음 임계값 도달

전환 도중 이후 페이즈의 체력 조건까지 충족하면 `ShouldPreemptTransitionForHealth()`로 확인한다. Character는 현재 전환 GA를 취소해 링, 몽타주 및 이동을 정리한다. 페이즈 확정은 State Tree의 Complete Task가 담당하며, 이후 현재 체력으로 다음 전환을 다시 요청한다.

### 페이즈 상태 복제와 이동 적용

현재 페이즈, 대기 페이즈 및 전환 상태는 복제한다. 클라이언트는 `OnRep_CurrentPhaseIndex()`의 변경 알림으로 Form을 갱신하며, 패턴 선택은 서버가 수행한다. Character는 페이즈 변경 시 이동 프로필, `bStationary`, `FacingMode`와 메시·캡슐 배율을 반영한다. 이동 오버라이드가 없으면 Definition의 기본 프로필을 사용한다.

## 관련 구현

### C++

- `Source/TeamProject_Fin/GameCore/Combat/BossPhaseComponent.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_BossPhaseTransitionBase.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_BossPhaseTransition.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Animation/AN_BossFormScale.h/.cpp`
- `Source/TeamProject_Fin/FSTC_BossPhaseTransitionInProgress.h/.cpp`
- `Source/TeamProject_Fin/FSTT_BeginBossPhaseTransition.h/.cpp`
- `Source/TeamProject_Fin/FSTT_ExecuteBossPhaseTransition.h/.cpp`
- `Source/TeamProject_Fin/FSTT_CompleteBossPhaseTransition.h/.cpp`

### Blueprint / Asset

- `Content/Monster/DA/Boss/BP_Boss_PhaseSet`
- `Content/Monster/DA/Boss/DA_Boss_Phase1`
- `Content/Monster/DA/Boss/DA_Boss_Phase2`
- `Content/Monster/DA/Boss/DA_Boss_Phase3`
- `Content/Monster/GAS/GA/BP_GA_BossPhase2Transition`
- `Content/Monster/GAS/GA/BP_GA_BossPhase3Transition`
- `Content/Monster/Anim/Montage/Boss/AM_Boss_Phase3Transition`
- `Content/Monster/Curve/Curve_BossPhase3Form`

## 사용 방법

1. 보스 Definition DA의 `BossPhaseSet`에 `BP_Boss_PhaseSet`을 지정한다.
2. `BP_Boss_PhaseSet` 배열을 P1, P2, P3 순서로 구성한다.
3. 각 페이즈에 체력 임계값, Pattern Set, 전환 GA 태그를 지정한다.
4. 보스 Ability Set에 P2/P3 전환 GA를 등록한다.
5. 전환 몽타주에 `Reposition`, `Rise`, 링 Spawn/Launch, Form Scale Notify를 배치한다.
6. `ST_Enemy_Boss`에 전환 Condition 및 Begin/Execute/Complete Task를 연결한다.
7. `BP_Boss_PhaseSet`의 P3 항목에서 Form 배율과 캡슐 배율을 설정한다. `DA_Boss_Phase3`는 해당 페이즈의 Pattern Set으로 참조한다.

## 테스트 및 확인 항목

개별 거리·확률·피해량·쿨다운 수치는 DA, BP 및 몽타주 설정에서 관리한다. 이 문서의 확인 항목은 실행 검증 체크리스트이며 완료 결과를 의미하지 않는다. Blueprint 내부 참조와 ST의 실제 전이는 에디터에서 확인한다.

- [ ] P1 시작 시 `DA_Boss_Phase1` Pattern Set이 활성화되는지 확인
- [ ] P2 체력 임계값에서 전환 GA가 한 번만 실행되는지 확인
- [ ] P3 체력 임계값에서 전환 GA가 한 번만 실행되는지 확인
- [ ] 전환 중 기존 공격 패턴이 취소되는지 확인
- [ ] 전환 종료 후 새 Pattern Set이 활성화되는지 확인
- [ ] Reposition/Rise 노티파이가 중복 실행되지 않는지 확인
- [ ] P2 링 버스트 Spawn 및 Launch 확인
- [ ] P3 재배치, 상승, 스케일 커브 적용 확인
- [ ] P3 영구 Form 배율이 메시와 캡슐에 적용되는지 확인
- [ ] 전환 중단 또는 보스 사망 후 전환 상태가 고정되지 않는지 확인

## 후속 작업

- 페이즈별 이동 프로필, 제자리 전투 및 바라보기 설정 검증
- 복제된 페이즈 인덱스로 클라이언트 Form이 갱신되는지 검증
- 보스 전용 체력바에 현재 페이즈 정보 표시
- 페이즈별 공격 패턴, VFX, SFX 추가
