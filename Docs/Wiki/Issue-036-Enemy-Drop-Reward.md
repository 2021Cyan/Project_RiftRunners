# #36 적 처치 보상 드랍 구현

## Issue

- Issue 번호: #36
- 기능명: 적 처치 보상 드랍 구현
- 담당자: horimihorimi
- 담당 영역: Combat / GAS / Replication / UI
- 관련 Sprint: Sprint 4
- 선행 이슈: #32 근접형 일반 적 구현, #35 적 체력·피격 및 사망 처리 구현

## 목적

몬스터를 처치했을 때 골드와 무기가 보상으로 지급되도록 구현한다.

골드는 파티 전체에 분배되는 재화이므로 누가 얼마나 기여했는지에 따라 나누고, 무기는 별도로 개발 중인 스폰 시스템에 위임한다. 이 문서의 범위는 보상 판정과 골드 지급까지이며, 무기의 실제 스폰은 연결부(델리게이트)만 제공한다.

## 보상 정책

| 보상 | 지급 방식 | 시점 | 확률 |
|---|---|---|---|
| 골드 | 기여도 비율로 분배 | 사망 확정 즉시 | 없음 (확정) |
| 무기 | 스폰 시스템에 요청 | 사망 연출 종료 후 | `WeaponDropChance` |

## 주요 구조

```text
피격 (모든 데미지)
        ↓
CombatDamageEventRouter::RouteDamageExecuted
        ├─ RecordDamage()      → 플레이어별 기여도 누적
        └─ bJustDied → NotifyKilled()
                          ↓
                  UMonsterDropComponent
                          ↓
              DropRow(FDataTableRowHandle) 조회
                          ↓
              생존 기여자 필터 → 비율 분배
                          ↓
              ABasePlayerState::AddGold()
                          ↓
              OnGoldChanged → WBP_GamePlayBase

사망 몽타주 종료
        ↓
AMonsterCharacter::FinishDeath → OnMonsterDeath
        ↓
UMonsterDropComponent::HandleMonsterDeath
        ↓
확률 판정 → OnWeaponDropRequested (무기 스폰 시스템)
```

## 설계 결정

### 골드 저장 위치: AttributeSet이 아닌 PlayerState

GAS 기반 프로젝트이므로 골드를 `UCharacterAttributeSet`에 두는 방안을 먼저 검토했으나 채택하지 않았다.

`AbilitySystemComponent`와 `CharacterAttributeSet`은 Pawn(`ABaseCharacter`)에 부착되어 있다. 그런데 이 프로젝트는 섹터를 이동할 때마다 `ServerTravel`을 수행하고, 이때 Pawn은 파괴 후 재생성된다. AttributeSet에 골드를 두면 섹터를 넘어갈 때마다 보유 골드가 0으로 초기화된다.

`ABasePlayerState`는 심리스 트래블을 넘어 값을 유지하는 `CopyProperties()` 경로가 이미 구현되어 있으므로, 골드를 여기에 두고 `CopyProperties()`에 이월 처리를 추가했다. 부수적으로 `FGameplayAttributeData`가 float 기반이라 정수 재화에 누적 오차가 생기는 문제, 그리고 골드가 Armor·AttackPower처럼 GameplayEffect 버프 대상으로 오인될 여지도 함께 회피된다.

### 기여도 집계 지점: 데미지 라우터

사망 처리 경로(`State.Dead` 태그 → `OnDeadTagChanged` → `BeginDeath` → `FinishDeath`)에는 가해자 정보가 실려 있지 않다. `OnMonsterDeath` 델리게이트도 죽은 몬스터만 전달한다. 따라서 사망 이후 시점에는 누가 때렸는지 알 방법이 없다.

`CombatDamageEventRouter::RouteDamageExecuted()`가 모든 데미지가 통과하는 단일 지점이고, 이곳에 `InstigatorActor`와 `AppliedDamage`, 사망 확정 여부(`bJustDied`)가 모두 존재한다. 기여도 집계와 지급 트리거를 이 지점에 배치했다.

막타 기준이 아니라 누적 기여도 기준을 택한 이유는, 막타는 폭발이나 도트 데미지에 좌우되어 실제 전투 기여와 무관하게 결정되는 경우가 많기 때문이다. 단일 집계 지점이 이미 확보되어 있어 누적 집계의 추가 비용도 크지 않다.

### 지급 시점: 사망 확정 순간

골드 지급을 `FinishDeath`(사망 몽타주 종료)가 아니라 사망이 확정되는 순간에 수행한다.

몽타주 종료까지 기다리면 그 사이 파티원의 생사가 바뀔 수 있어, 사망 시점의 생존자 명단을 별도로 스냅샷해 두어야 한다. 사망 순간에 즉시 지급하면 이 문제가 발생하지 않으며, UI 반응도 처치와 동시에 이루어진다.

무기는 시체 위치에 스폰되는 것이 자연스러우므로 연출 종료 후로 분리했다.

### 컴포넌트 분리

드랍 로직을 `AMonsterCharacter`에 직접 넣지 않고 `UMonsterDropComponent`로 분리했다. `AMonsterCharacter`는 이미 AI·피격·등장 처리를 담당하고 있어 보상 로직까지 포함하면 책임이 과도해지고, 보스나 특수 몬스터의 예외 처리를 위해 virtual 오버라이드가 늘어난다.

컴포넌트는 `AMonsterCharacter` 생성자에서 기본 부착한다(`MotionWarpingComponent`, `CombatPositioningComponent`와 동일한 방식). 이로써 몬스터 Blueprint마다 컴포넌트를 추가하는 수작업 없이, Blueprint별 보상 설정만 지정하면 된다.

### 드랍 테이블 참조 방식

`DropRow`는 `FDataTableRowHandle`로 선언하고 `meta = (RowType = ...)`를 지정했다. `ABaseCharacter::InitStatsRow`가 사용하는 방식과 동일하다. 에디터에서 RowType이 일치하는 DataTable만 후보로 표시되므로 잘못된 테이블 지정을 방지할 수 있다.

몬스터 GameplayTag로 행을 자동 조회하는 방식도 검토했으나, 태그가 일치하지 않을 때 보상이 조용히 누락되는 경로가 생기므로 명시적 지정을 택했다.

## 구현 내용

### 골드 보관 및 입출금 (`ABasePlayerState`)

```cpp
UPROPERTY(ReplicatedUsing = OnRep_Gold)
int32 Gold; // 초기값은 PlayerState 기본 설정을 사용한다.

void AddGold(int32 Amount);          // 서버 전용
bool TrySpendGold(int32 Cost);       // 서버 전용
int32 GetGold() const;               // BlueprintPure
bool CanAfford(int32 Cost) const;    // BlueprintPure

FOnGoldChanged OnGoldChanged;        // (NewGold, Delta)
```

- `AddGold` / `TrySpendGold`는 `HasAuthority()` 검사를 통과해야 동작한다. 클라이언트 호출은 조용히 무시된다.
- `TrySpendGold`는 잔액이 부족하면 아무것도 차감하지 않고 `false`를 반환한다. 물품 지급은 호출부 책임이므로 반환값을 반드시 확인해야 한다.
- `AddGold`는 누적값이 `int32` 상한을 넘어 음수로 뒤집히지 않도록 상한에서 잘라낸다.
- `CopyProperties()`에 `Gold` 이월을 추가해 섹터 이동 시 보유 골드를 유지한다.
- `OnGoldChanged`는 서버에서는 `AddGold` 내부에서, 클라이언트에서는 `OnRep_Gold`에서 발화한다.
- 접속·트래블 직후의 첫 복제는 획득이 아니라 초기 동기화이므로 `Delta`를 0으로 억제한다. 그렇지 않으면 접속하자마자 누적 골드 전액에 대한 획득 연출이 재생된다.

### 드랍 테이블 (`FMonsterDropRow`)

| 필드 | 타입 | 설명 |
|---|---|---|
| `GoldReward` | `int32` | 파티에 분배할 총 골드 |
| `GoldVariance` | `float` | 보상 총량에 적용할 변동 폭 |
| `WeaponDropChance` | `float` | 무기 드랍 요청 확률 |

### 기여도 집계 (`UMonsterDropComponent::RecordDamage`)

- 모든 피격에서 호출되며, 사망타의 피해량도 포함해 누적한다.
- 누적 기준은 `LocalDamageDone`이 아니라 `AppliedDamage`다. 남은 체력을 초과한 오버킬 피해가 기여도를 부풀리지 않도록 하기 위함이다.
- Instigator에서 `ABasePlayerState`를 역추적한다. Pawn, Controller, PlayerState, 투사체의 Instigator까지 순서대로 확인한다.
- PlayerState를 찾지 못하면 집계에서 제외된다. 몬스터 자폭, 몬스터 간 팀킬, 환경 데미지가 여기에 해당한다.
- 지급 완료 후 들어오는 시체 타격은 집계하지 않는다.

### 골드 분배 (`UMonsterDropComponent::PayOutGold`)

```text
1. 기여자 중 생존자만 남긴다
2. TotalWeight = 생존 기여자들의 기여도 합
3. 기여도 내림차순 정렬
4. 개인 지급액 = Floor(TotalGold × 개인기여도 / TotalWeight)
5. 내림으로 남은 잔돈은 최대 기여자에게 지급
```

- 사망자 방어: 사망한 기여자는 목록에서 제외한다. `TotalWeight`를 생존자만으로 계산하므로 사망자의 몫은 소멸하지 않고 생존자에게 자동 재분배된다.
- 생존 판정은 Pawn의 ASC가 `State.Dead` 태그를 보유하는지로 확인하고, Pawn이나 ASC를 얻을 수 없으면 PlayerState의 체력 사본으로 판정한다.
- 잔돈 처리로 총 지급액이 변동 폭까지 반영한 실제 보상 총량과 일치한다.
- 생존 기여자가 한 명도 없으면 지급하지 않는다.

### 무기 드랍 연결부

이 컴포넌트는 무기 드랍 확률을 판정하고 실제 스폰 요청을 델리게이트로 전달한다.

```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(
    FOnWeaponDropRequested,
    AMonsterCharacter*, DeadMonster,
    FTransform, DropTransform);

UPROPERTY(BlueprintAssignable, Category = "Drop")
FOnWeaponDropRequested OnWeaponDropRequested;
```

서버 확인, 행 조회, 확률 판정을 모두 통과한 경우에만 발화한다. 수신 측은 조건 검사 없이 스폰만 수행하면 된다.

`AMonsterCharacter::FinishDeath()`는 일반 몬스터를 `CorpseLifeSpan` 후 제거한다. 풀링 몬스터는 스폰 포인트가 풀로 반환한다. 드랍 액터는 시체 수명과 분리해 월드에 독립적으로 스폰한다.

C++에서 확인되는 연결부는 `OnWeaponDropRequested`의 Broadcast다. 실제 무기 생성 여부와 Blueprint 수신 연결은 에디터에서 확인한다.

### UI 연동

`WBP_GamePlayBase`(`UBaseWidgetLayer` 파생)에서 `OnGoldChanged`에 바인딩한다. 이 레이어는 씬당 하나만 존재하며 ESC로 닫히지 않으므로 상시 표시되는 재화 표기에 적합하다.

바인딩 직후 `GetGold()`로 현재값을 한 번 반영해야 한다. 델리게이트는 값이 변할 때만 발화하므로, 이미 누적된 골드는 바인딩만으로 표시되지 않는다.

골드 표시와 획득 연출의 실제 배치는 UI Blueprint에서 확인한다. 코드에서는 초기값 반영과 변경 알림을 구분한다.

### 로그

`[MonsterDrop]` 접두어로 지급 전 과정을 출력한다.

```text
[MonsterDrop] 지급 시작 | Monster=... | Row=... | TotalGold=... | 대상=N명 | 유효기여도=... | 제외기여도=...
[MonsterDrop]   PS=... | Nickname=... | Damage=... (xx.x%) | 지급=... | 보유=...
[MonsterDrop]   잔돈 N -> 최대기여자 PS=... | 보유=...
[MonsterDrop] 제외(사망) | Monster=... | PS=... | Damage=...
[MonsterDrop] 지급 대상 없음 | Monster=... | TotalGold=... | 사망자기여도=...
[MonsterDrop] 무기 드랍 요청 | Monster=... | Chance=... | Listeners=N
```

`Listeners=0`은 확률에는 당첨되었으나 `OnWeaponDropRequested`에 바인딩된 수신 측이 없다는 의미다.

## 관련 구현

### C++

- `Source/TeamProject_Fin/GameCore/BasePlayerState.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Combat/MonsterDropComponent.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Combat/CombatDamageEventRouter.cpp`
- `Source/TeamProject_Fin/GameCore/Character/MonsterCharacter.h/.cpp`

### Blueprint / Asset

- `Content/Monster/DT/DT_MonsterDropGold`
- `Content/TPSDevelop/UI/GamePlay/WBP_GamePlayBase`
- 각 몬스터 Blueprint의 `DropComponent > Drop Row`

## 사용 방법

1. `DT_MonsterDropGold`에 몬스터 종류별 행을 추가하고 `GoldReward`를 지정한다. `GoldVariance`는 보상 변동 정책에 맞게 설정한다.
2. 몬스터 Blueprint의 `DropComponent > Drop Row`에 DataTable과 행 이름을 지정한다. 지정하지 않으면 해당 몬스터는 보상을 지급하지 않는다.
3. 무기 스폰 시스템은 `UMonsterDropComponent::OnWeaponDropRequested`에 바인딩하고, 스폰한 액터를 몬스터에 어태치하지 않는다.
4. UI 위젯에서 `ABasePlayerState::OnGoldChanged`에 바인딩하고, 바인딩 직후 `GetGold()`로 초기값을 반영한다.
5. 상점 등 소비 지점에서는 `TrySpendGold()`의 반환값을 확인한 뒤 물품을 지급한다.

> DataTable에서 행 이름을 변경하면 Blueprint의 `Drop Row`는 자동으로 따라가지 않는다. 행 이름을 바꾼 경우 해당 행을 참조하는 모든 몬스터 Blueprint를 다시 지정해야 한다.

## 테스트 및 확인 항목

개별 거리·확률·피해량·쿨다운 수치는 DA, BP 및 몽타주 설정에서 관리한다. 이 문서의 확인 항목은 실행 검증 체크리스트이며 완료 결과를 의미하지 않는다. Blueprint 내부 참조와 ST의 실제 전이는 에디터에서 확인한다.

- [x] `DropRow` 지정 후 몬스터 처치 시 골드가 지급되는지 확인
- [x] 단독 처치 시 `GoldReward` 전액이 지급되는지 확인
- [x] 지급 로그의 기여도 비율과 지급액이 일치하는지 확인
- [x] `DropRow` 미지정 몬스터가 보상을 지급하지 않는지 확인
- [x] 골드 보유량이 UI에 표시되고 획득 시 갱신되는지 확인
- [ ] 2인 플레이에서 기여도 비율대로 분배되는지 확인
- [ ] 사망한 기여자가 제외되고 몫이 생존자에게 재분배되는지 확인
- [ ] 잔돈이 최대 기여자에게 지급되어 총액이 일치하는지 확인
- [ ] 몬스터 자폭·팀킬로 사망한 경우 지급되지 않는지 확인
- [ ] 섹터 이동 후 보유 골드가 유지되는지 확인
- [ ] `WeaponDropChance` 확률대로 드랍 요청이 발생하는지 확인

## 후속 작업

### 무기 드랍 구현

현재는 확률 판정과 `OnWeaponDropRequested` 델리게이트까지만 구현되어 있다. 실제 스폰은 미구현 상태다.

- 델리게이트를 수신해 무기를 스폰하는 처리 구현
- 소유권 정책 확정. 공용 드랍 1개로 뿌릴지, 기여자별 개인 드랍으로 지급할지 결정이 필요하다. 기존 `AGunObject::SpawnRandomPersonalDrop()`은 소유 PlayerController를 받는 개인 드랍이며, 현재 델리게이트는 PlayerController를 전달하지 않는다. 개인 드랍으로 갈 경우 델리게이트에 기여자 목록을 추가해야 한다.
- 스폰 위치 보정. `DropTransform`은 몬스터의 액터 트랜스폼이므로 발밑 기준이다. 바닥에 파묻히지 않도록 수신 측에서 오프셋을 적용해야 한다.
- 몬스터별 `WeaponDropChance` 값 설정

### 골드 UI 에셋 연결

현재 표기는 값 검증용 테스트 구현이다.

- 골드 아이콘, 배경, 폰트 등 HUD 디자인 에셋 연결
- 획득 시 `Delta` 기반 획득량 표시 연출
- 골드 부족 시 안내 표기
- HUD 레이아웃 내 배치 확정

### 그 외

- 상점 가격 확정 후 `GoldReward` 수치 밸런싱
- 상점 결제 경로에 `TrySpendGold()` 연동
- 사망 시 골드 손실 등 페널티 정책 검토
- 골드/무기 외 보상 종류 확장 시 드랍 테이블 확장
