# #39 Alepsis-Taura 보스 공격 패턴 구현

## Issue

- Issue 번호: #39
- 기능명: Alepsis-Taura 보스 공격 패턴 구현
- 담당 영역: Boss Combat / GAS / State Tree / Animation / Projectile
- 관련 Sprint: Sprint 6
- 선행 이슈: #37 보스 기본 캐릭터 구현, #38 보스 페이즈 전환 시스템 구현

## 목적

Alepsis-Taura가 현재 페이즈의 Pattern Set에서 공격을 선택하고, Gameplay Ability와 몽타주 Notify를 통해 근접 공격, 거울 파동, 원형 링, 기둥, 유도 미사일을 실행하도록 구현한다.

패턴 선택은 State Tree가 담당하고, 공격마다 필요한 판정 및 생성 로직은 GA와 복제 Actor에 분리한다. 이 구조로 공격 연출은 모든 클라이언트에 표시하면서 실제 피해 처리는 서버에서만 수행한다.

## 전체 흐름

```text
보스 조우 연출 완료
    ↓
현재 Phase Pattern Set 활성화
    ↓
State Tree가 거리 / 쿨다운 / 체력 조건으로 패턴 선택
    ↓
Ability Tag로 Gameplay Ability 활성화
    ↓
공격 몽타주 재생
    ↓
Anim Notify가 Gameplay Event 전달
    ├─ 근접 공격: HitWindow Trace
    ├─ 거울: Wave Projectile 생성
    ├─ 링: Ring Wave 생성·발사
    ├─ 기둥: Pillar 생성 및 Slow GE 적용
    └─ 미사일: 대상 플레이어 추적 발사
    ↓
GA 종료 및 Recovery 대기
    ↓
다음 패턴 선택
```

## 공통 처리

### 패턴 선택과 실행

- `FSTT_SelectEnemyPattern`이 활성 Pattern Set에서 실행 가능한 패턴을 선택한다.
- `FSTT_PrepareEnemyPattern`이 패턴 실행 전 필요한 위치 보정을 처리한다.
- `FSTT_ExecuteEnemyPattern`이 선택된 Ability Tag로 GA를 실행한다.
- `FSTT_WaitPatternRecovery`가 공격 종료 후 다음 패턴까지의 대기 시간을 처리한다.
- 페이즈 전환이 요청되면 현재 Pattern Ability와 준비 이동을 취소한 뒤 전환 State로 이동한다.

### 실행 태그와 피해 태그

Pattern Set의 `AbilityTag`는 `Ability.Enemy.Boss.*` 실행 태그를 사용한다. 피해 테이블 조회용 `Action.AI.Boss.*` 태그와 구분한다. 예를 들어 2연속 휘두르기는 `Ability.Enemy.Boss.ForwardDoubleSwing`으로 실행하고 기본 피해 태그는 `Action.AI.Boss.ForwardDoubleSwing`을 사용한다.

### 서버 판정과 클라이언트 연출

- 근접 HitWindow, 파동 투사체, 링, 기둥, 미사일의 피해 적용은 서버만 수행한다.
- `BossWaveProjectile`, `BossRingWave`, `BossP3AttackPillar`는 복제 Actor이므로 클라이언트에도 공격 위치와 연출 상태가 전달된다.
- HitWindow는 클라이언트에서도 Trace 결과와 빔 같은 시각 연출의 끝점을 갱신한다. GameplayEffect 피해 적용만 서버 권한으로 제한한다.

### 공격 중단 정리

- `GA_BossProjectileRings`는 자신이 생성한 모든 링을 종료 시 제거한다.
- `GA_BossGroundSlam`은 활성 지면 강타 링을 종료 시 제거한다.
- `GA_BossPhaseTransition`은 전환 중 생성한 모든 링을 종료 시 제거한다.
- `BossP3AttackPillar`은 파괴 또는 수명 종료 시 적용 중인 Slow GE를 모두 회수한다.

## Phase 1 공격 패턴

Phase 1은 공중 이동 형태에서 일반 Pattern Set으로 공격을 선택한다.

### 거울 발사 — `GA_BossShatteredMirror`

보스 메시의 거울 본 위치에서 직사각형 거울 파동을 직선 발사한다.

1. 몽타주의 `Event.AI.Boss.ShatteredMirror.Fire` 이벤트를 수신한다.
2. `WaveEmitters`에 등록된 각 거울 본의 월드 Transform을 읽는다.
3. 본의 X/Y/Z 축 중 GA 설정 축을 발사 방향으로 정한다.
4. 본마다 `BossWaveProjectile`을 생성한다.
5. 파동은 Box Sweep으로 이동 경로 전체를 검사하며, 다른 Actor에 닿아도 멈추지 않는다.
6. 플레이어 태그를 가진 대상에게 파동 하나당 한 번만 Damage GE를 적용한다.

동일한 `GA_BossShatteredMirror`는 Phase 2 Pattern Set에서도 재사용한다.

### 전방 2연속 휘두르기 — `GA_BossForwardDoubleSwing`

보스 정면 근거리에서 연속으로 두 번 휘두르는 공격이다.

- 공격 태그: `Action.AI.Boss.ForwardDoubleSwing`
- `GA_BossForwardDoubleSwing`은 공통 Pattern Base를 통해 공격 몽타주를 재생한다.
- 실제 휘두르기 시점, 범위, 각 타격의 피해는 몽타주의 HitWindow Notify가 처리한다.

### 원형 링 빔 — `GA_BossProjectileRings`

보스 중심에서 원형 띠 형태의 빔을 생성한 뒤 외곽으로 확장 발사한다.

1. `Event.AI.Boss.RingSpawn` 이벤트마다 `BossRingWave`를 하나 생성한다.
2. 첫 링은 `InitialPlane` 설정을 사용한다.
3. 이후 링은 수평 평면과 수직 평면을 번갈아 사용한다.
4. `Event.AI.Boss.RingLaunch` 이벤트가 오면 마지막 생성 링이 최대 반경까지 확장한다.
5. 링은 원형 띠에 닿은 플레이어만 검사하며, 확장 중 같은 플레이어에게 한 번만 피해를 준다.
6. 최대 반경에 도달한 링은 제거된다.

### 지면 강타 — `GA_BossGroundSlam`

보스가 지면을 내려찍고, 중심에서 수평 링을 확장한 뒤 다시 중심으로 수축시킨다.

1. `Event.AI.Boss.GroundSlam.Impact` 이벤트에서 보스 주변 원형 범위에 즉시 피해를 한 번 적용한다.
2. 같은 이벤트에서 `BossRingWave`를 생성한다.
3. 링은 보스 중심에서 최대 반경까지 확장한다.
4. `Event.AI.Boss.GroundSlam.Retract` 이벤트가 오면 링이 중심으로 수축한다.
5. 확장 판정과 수축 판정은 별도로 기록하므로, 같은 플레이어는 확장 시 한 번, 수축 시 한 번 피해를 받을 수 있다.

## Phase 2 전환 및 공격 패턴

### Phase 1 → Phase 2 전환 공격 — `GA_BossPhaseTransition`

Phase 2 전환은 외형 교체만 수행하지 않고, 중앙 재배치와 링 버스트 공격을 포함한다.

1. 전환 GA가 보스를 아레나 앵커 기준 위치로 재배치한다.
2. `Rise` 이벤트에서 지정된 앵커 오프셋까지 전환 이동을 시작한다.
3. 이동 완료 후 몽타주를 Ring Burst 구간으로 연결한다.
4. `RingSpawn` 이벤트마다 현재 `RingBursts` 항목의 링 묶음을 생성한다.
5. 버스트는 수평 링, 타깃 방향 수직 링, 타깃 방향 측면 수직 링을 조합할 수 있다.
6. `RingLaunch` 이벤트에서 현재 버스트의 모든 링을 동시에 발사한다.
7. 모든 버스트가 끝나면 다음 전환 몽타주 섹션으로 진행한다.
8. 전환이 끝나면 Phase 2 Pattern Set과 Movement Profile을 적용한다.

### 거울 발사 — `GA_BossShatteredMirror`

Phase 1과 동일하게 거울 본에서 파동 투사체를 생성해 직선 발사한다.

### 촉수 비틀기 — `GA_BossWhackTwist`

보스가 아레나 중심을 기준으로 Root Motion 위치를 보정하며 촉수 공격을 수행한다.

1. 공격 시작 전 `ArenaCenter`를 Motion Warping Target으로 등록한다.
2. 공격 몽타주의 Root Motion을 등록된 Target 위치 기준으로 보정한다.
3. 몽타주의 HitWindow Notify가 촉수 타격 범위와 피해 시점을 처리한다.
4. GA 종료 또는 취소 시 등록한 Motion Warping Target을 제거한다.

### 촉수 뻗기 — `GA_BossTendrilHorrorEject`

공중에서 촉수를 플레이어 방향으로 뻗는 공격에 사용하는 Pattern GA다.

- 공격 태그: `Action.AI.Boss.TendrilHorrorEject`
- GA는 공통 Pattern Base를 사용해 공격 몽타주를 재생한다.
- 촉수의 실제 타격 시점과 범위는 몽타주의 HitWindow Notify 설정으로 처리한다.

### 촉수 강타 — `BP_GA_BossTendrilSlam`

촉수의 지면 강타와 후속 휘두르기 공격은 BP GA와 공격 몽타주로 구성한다.

- 패턴 선택은 Phase 2 Pattern Set에서 담당한다.
- 몽타주 Notify가 공격 이벤트와 피해 판정 시점을 결정한다.
- 공격 연속 횟수와 각 타격의 범위는 BP 및 몽타주 설정값으로 조정한다.

## Phase 3 전환 및 공격 패턴

### Phase 2 → Phase 3 전환 — `BP_GA_BossPhase3Transition`

Phase 3 전환은 공통 전환 GA 기반으로 중앙 재배치, 상승 이동, 전환 몽타주 재생을 수행한다.

- `AN_BossFormScale` Notify State가 전환 중 스케일 커브를 적용한다.
- 전환 완료 후 Phase 3의 메시와 캡슐 배율을 유지한다.
- 전환 완료 시 Phase 3 Pattern Set을 활성화한다.

### Phase 3 복합 공격 — `GA_BossP3Attack`

Phase 3의 거대 거울, 슬로우 기둥, 유도 미사일, 샤우트는 하나의 GA가 몽타주 이벤트 순서에 맞춰 실행한다.

#### 거대 사선 거울

1. `Event.AI.Boss.P3Attack.MirrorFire` 이벤트에서 `BossWaveProjectile` 기반 거울을 생성한다.
2. 거울은 보스 전방 위쪽에서 생성되고, 설정된 Pitch만큼 아래로 기울어진 방향으로 이동한다.
3. 거울의 Box Extent를 기준으로 플레이어를 관통하며, 대상당 한 번만 피해를 준다.
4. 거울의 진행 방향은 이후 기둥 배치 방향으로 저장한다.

#### 슬로우 기둥

1. `Event.AI.Boss.P3Attack.PillarSpawn` 이벤트에서 거울 위치를 기준으로 기둥을 생성한다.
2. 기본 오프셋은 거울 기준 좌측·중앙·우측이며, 기둥 수와 배치는 `PillarSpawnOffsets`로 조정한다.
3. 각 기둥은 시작 지점에서 거울 진행 방향으로 바닥을 Trace해 지면까지 연결된다.
4. 기둥 내부 플레이어에게 Infinite Slow GE를 적용한다.
5. 플레이어가 기둥 밖으로 나가거나 기둥이 파괴되면 적용한 Slow GE를 제거한다.

#### 유도 미사일

1. `Event.AI.Projectile.Fire` 이벤트마다 미사일 버스트를 실행한다.
2. 기본 버스트는 보스 주변 서로 다른 높이와 좌우 위치에서 설정된 발사 위치마다 미사일을 생성한다.
3. 범위 안의 살아 있는 플레이어를 수집한다.
4. 미사일마다 대상 플레이어를 순환 배정하고 `LaunchAtTarget`으로 유도 발사한다.
5. 미사일 피해는 P3 Attack에 설정한 공격 태그와 Damage GE를 사용한다.

#### 샤우트와 경직

1. `Event.AI.Boss.P3Attack.Shout` 이벤트에서 샤우트 판정을 한 번 수행한다.
2. `ShoutChance` 확률에 성공하면 범위 안의 살아 있는 모든 플레이어에게 Stagger GE를 적용한다.
3. 적용한 GE Handle을 보관한다.
4. 공격이 취소되면 보관 중인 Handle을 즉시 제거한다.

## 관련 구현

### C++

- `Source/TeamProject_Fin/FSTT_SelectEnemyPattern.h/.cpp`
- `Source/TeamProject_Fin/FSTT_PrepareEnemyPattern.h/.cpp`
- `Source/TeamProject_Fin/FSTT_ExecuteEnemyPattern.h/.cpp`
- `Source/TeamProject_Fin/FSTT_WaitPatternRecovery.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_BossShatteredMirror.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_BossForwardDoubleSwing.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_BossProjectileRings.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_BossGroundSlam.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_BossWhackTwist.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_BossTendrilHorrorEject.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_BossP3Attack.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_BossPhaseTransitionBase.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_BossPhaseTransition.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Projectile/BossWaveProjectile.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Projectile/BossRingWave.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Projectile/BossP3AttackPillar.h/.cpp`

### Blueprint / Asset

- `Content/Monster/AI/ST/ST_Enemy_Boss`
- `Content/Monster/DA/Boss/DA_Boss_Phase1`
- `Content/Monster/DA/Boss/DA_Boss_Phase2`
- `Content/Monster/DA/Boss/DA_Boss_Phase3`
- `Content/Monster/GAS/GA/BP_GA_BossPhase2Transition`
- `Content/Monster/GAS/GA/BP_GA_BossPhase3Transition`
- `Content/Monster/GAS/GA/BP_GA_BossTendrilSlam`
- `Content/Monster/Anim/Montage/Boss`

## 테스트 및 확인 항목

개별 거리·확률·피해량·쿨다운 수치는 DA, BP 및 몽타주 설정에서 관리한다. 이 문서의 확인 항목은 실행 검증 체크리스트이며 완료 결과를 의미하지 않는다. Blueprint 내부 참조와 ST의 실제 전이는 에디터에서 확인한다.

- [ ] Phase 1 Pattern Set에서 거울, 2연속 휘두르기, 링 빔, 지면 강타가 정상 선택되는지 확인
- [ ] 거울 파동이 플레이어를 관통하며 대상당 한 번만 피해를 주는지 확인
- [ ] 링의 수평·수직 평면과 확장·수축 피해가 정상인지 확인
- [ ] Phase 2 전환 중 Ring Burst가 설정된 방향과 횟수대로 생성·발사되는지 확인
- [ ] Phase 2에서 거울, 촉수 비틀기, 촉수 뻗기, 촉수 강타가 정상 실행되는지 확인
- [ ] Phase 3 거대 거울이 기울어진 방향으로 이동하는지 확인
- [ ] Phase 3 기둥 내부에서만 Slow GE가 유지되고 이탈·파괴 시 해제되는지 확인
- [ ] Phase 3 미사일이 살아 있는 플레이어를 추적하는지 확인
- [ ] 샤우트 경직이 확률과 범위 설정대로 적용·해제되는지 확인
- [ ] 공격 또는 전환 취소 후 링·기둥·지속 GE가 남지 않는지 확인
