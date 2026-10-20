# #40 보스 사망 및 게임 클리어 처리 구현

## Issue

- Issue 번호: #40
- 기능명: 보스 사망 및 게임 클리어 처리 구현
- 담당자: horimihorimi
- 담당 영역: Boss Combat / GAS / Event / UI / Multiplayer
- 관련 Sprint: Sprint 6
- 선행 이슈: #37 보스 기본 캐릭터 구현, #39 보스 공격 패턴 구현

## 목적

보스 처치 시 사망 처리를 수행하고, 게임 클리어 상태와 클리어 알림을 모든 플레이어에게 전달한다.

보스 사망, 클리어 연출, 결과 화면 표시를 분리한다. 사망 시작 시 전투 통계를 확정해 연출 대기 시간이 클리어 기록에 포함되지 않도록 하고, 결과 확인 후 전원이 준비하면 파티 단위로 대기방에 복귀한다.

## 전체 흐름

```text
보스 체력 0 이하
    ↓
마지막 타격 피해량 및 처치 기록
    ↓
Event.Death → GA_Death → Character.BeginDeath()
    ↓
OnDeathStarted → RequestBossEncounterEvent
    ├─ 통계 및 클리어 시간 확정
    └─ PostEvents 실행
        ↓
GameClearEvent → GameState.SetGameCleared()
    ↓
MulticastGameClear → 각 머신의 EventManager.OnGameClear
    ↓
클리어 연출 및 UI 처리
```

```text
클리어 후 결과 표시 이벤트
    ↓
ShowClearResultsEvent → GameState.ShowClearResults()
    ↓
MulticastShowClearResults → 로컬 PlayerController의 결과 UI
    ↓
플레이어별 복귀 준비 요청 → 서버의 전원 준비 확인
    ↓
GameMode.ReturnPartyToRoom()
    ├─ 파티 단위 대기방 이동
    ├─ 런 데이터 초기화
    └─ 대기방 세션 설정 복원
```

## 설계 결정

### 사망 시작과 사망 완료 분리

클리어 흐름의 시작점은 사망 몽타주 종료가 아닌 `OnDeathStarted`로 설정한다. `BeginDeath()`에서 사망 시작을 확정하고, `GA_Death`가 다른 Ability를 취소한 뒤 사망 몽타주를 재생한다. 몽타주 완료·중단·취소 시 사망 완료 경로를 처리하며, 몽타주가 없어도 사망 처리를 완료한다.

### 통계 확정과 클리어 연출 분리

`ARequestBossEncounterEvent::HandleBossDeathStarted()`에서 통계를 먼저 확정하고 `PostEvents`를 실행한다. 클리어 연출에 Delay를 추가해도 기록된 클리어 시간은 늘어나지 않는다.

### 서버 판정과 로컬 UI 분리

클리어 상태, 결과 표시 요청, 복귀 준비 판정은 서버에서 처리한다. UI 알림은 GameState의 Reliable Multicast를 통해 각 머신의 EventManager로 전달한다. 결과 위젯은 로컬 PlayerController만 생성한다.

### 클리어 알림과 결과 화면 분리

- `GameClearEvent`: 클리어 상태 확정 및 클리어 알림
- `ShowClearResultsEvent`: 클리어 이후 상세 결과 화면 표시

결과 표시 이벤트를 클리어 후 포탈 등 원하는 상호작용에 연결할 수 있도록 구성한다.

## 구현 내용

### 보스 사망 처리

`CombatDamageEventRouter`에서 새 체력이 0 이하가 되는 순간을 사망으로 판정한다.

1. 실제 적용 피해량과 마지막 타격의 처치 수를 기록한다.
2. 대상 ASC에 `Event.Death`를 전달한다.
3. `GA_Death`가 캐릭터의 `BeginDeath()`를 호출한다.
4. 이동을 정지하고 충돌 응답을 변경한다.
5. 다른 Ability를 취소하고 사망 연출을 실행한다.

`ABaseCharacter`의 `bDeathStarted`, `bDeathFinished`로 사망 시작과 완료의 중복 처리를 방지한다.

### 보스 조우 종료

`ARequestBossEncounterEvent`는 서버에서 조우를 요청할 때 보스의 `OnDeathStarted`를 구독한다. 사망 알림을 받으면 구독을 해제하고 `FinalizeRunStatistics()` 호출 후 `PostEvents`를 실행한다. 클리어 연출 순서와 Delay는 PostEvents에서 구성한다.

### 게임 클리어 상태

`UGameClearEvent`는 서버의 `ABaseGameState::SetGameCleared()`를 호출한다. 서버 권한과 중복 클리어 여부를 확인한 뒤 통계를 확정하고 `bGameCleared`를 설정한다. `MulticastGameClear()`는 각 머신의 `EventManager.OnGameClear`를 호출한다.

`bGameCleared`는 복제하며, `GameOverEvent`는 이미 클리어된 경우 게임오버 처리를 건너뛴다.

### 전투 통계

플레이어별 통계는 PlayerState의 `FRunStatistics`에 저장하고 복제한다.

| 필드 | 설명 |
|---|---|
| `TotalDamage` | 적에게 실제 적용한 누적 피해량 |
| `BossDamage` | 보스에게 실제 적용한 누적 피해량 |
| `KillCount` | 처치 수 |
| `DamageTaken` | 실제 받은 누적 피해량 |
| `ReviveCount` | 팀원 부활 횟수 |
| `ClearDurationSeconds` | 런 시작부터 보스 처치까지의 시간 |
| `bFinalized` | 최종 결과 확정 여부 |

피해량은 CombatDamageEventRouter, 팀원 부활 횟수는 부활 처리 경로에서 기록한다. 서버에서 통계 수집 중일 때만 집계하며 확정 이후에는 변경하지 않는다. 마지막 타격과 처치는 사망 이벤트 전달 전에 기록해 최종 결과에서 누락되지 않도록 한다.

### 클리어 시간

`APlayGameMode::BeginPlay()`에서 `URunStatisticsSubsystem::StartIfNeeded()`를 호출한다. GameInstance Subsystem에 시작 시간을 보관하므로 월드 전환 중에도 시간을 유지한다. `FPlatformTime::Seconds()`로 경과 시간을 계산하고 보스 사망 시작 시 확정한다. 새 런 시작 또는 복귀 시 측정 상태를 초기화한다.

### 결과 화면

`ShowClearResults()`는 서버 권한, 클리어 상태, 결과 표시 중복 여부를 확인한다. 알림을 받은 로컬 PlayerController는 `ClearResultsWidgetClass`로 위젯을 생성한다.

- 기존 기본 UI 숨김 및 경로 UI 닫기
- 캐릭터 이동 정지 및 이동·시점 입력 차단
- UI 전용 입력 모드와 마우스 커서 활성화
- 통계 변경 및 복귀 준비 상태 Delegate 연결

최종 통계가 아직 복제되지 않았다면 확정된 데이터가 도착한 뒤 결과 화면을 갱신한다.

### 파티 대기방 복귀

복귀 버튼은 소유 PlayerController의 서버 RPC로 준비 요청을 전달한다. 서버는 현재 연결된 플레이어인지 확인하고 준비 목록에 추가한다.

| 상태 | 역할 |
|---|---|
| `ReadyPlayers` | 준비 완료 플레이어 목록 |
| `PlayerCount` | 현재 복귀 대상 인원 |
| `bTravelStarted` | 대기방 이동 시작 여부 |

중복 요청은 `AddUnique()`로 방지한다. 퇴장한 플레이어는 대상과 준비 목록에서 제외한다. 남은 전원이 준비하면 이동을 한 번만 시작하며, 이동 예약 실패 시 준비 상태를 초기화한다.

`ReturnPartyToRoom()`은 기억된 대기방 맵 또는 기본 복귀 맵을 사용한다. 이동 예약 성공 후 골드, 전투 상태, 생성 무기 기록, 뽑기 누적 상태, 통계 및 월드 생성 런 상태를 초기화한다. 이름, 영웅 선택, 입장 순서는 유지한다. 복귀한 LobbyPlayerController는 대기방 UI를 표시하고 세션을 다시 참가 가능한 상태로 복원한다.

## 관련 구현

### C++

- `Source/TeamProject_Fin/GameCore/Ability/GA_Death.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Character/BaseCharacter.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Combat/CombatDamageEventRouter.cpp`
- `Source/TeamProject_Fin/GameCore/BaseGameState.h/.cpp`
- `Source/TeamProject_Fin/GameCore/BasePlayerState.h/.cpp`
- `Source/TeamProject_Fin/GameCore/BasePlayerController.h/.cpp`
- `Source/TeamProject_Fin/GameCore/BaseGameMode.h/.cpp`
- `Source/TeamProject_Fin/GameCore/PlayGameMode.cpp`
- `Source/TeamProject_Fin/GameCore/Match/RunStatistics.h`
- `Source/TeamProject_Fin/GameCore/Match/RunStatisticsSubsystem.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Match/PartyRoomSubsystem.h`
- `Source/TeamProject_Fin/GameCore/Match/LobbyPlayerController.h/.cpp`
- `Source/TeamProject_Fin/GameCore/UI/ClearResultsWidget.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Interact/ReviveZoneActor.h/.cpp`
- `Source/TeamProject_Fin/PCGMap/MapBuild/SectorBuild/Event/EventManager.h`
- `Source/TeamProject_Fin/PCGMap/MapBuild/SectorBuild/Event/PlacedEvent/RequestBossEncounterEvent.h/.cpp`
- `Source/TeamProject_Fin/PCGMap/MapBuild/SectorBuild/Event/ConceptEvent/GameClearEvent.h/.cpp`
- `Source/TeamProject_Fin/PCGMap/MapBuild/SectorBuild/Event/ConceptEvent/ShowClearResultsEvent.h/.cpp`
- `Source/TeamProject_Fin/PCGMap/MapBuild/WorldGenSubSystem.h/.cpp`

### Blueprint / Asset

- `Content/Monster/DA/BP_GameClear`
- `Content/Monster/DA/BP_ShowClearResults`
- `Content/Monster/UI/WBP_GameClear`
- `Content/Monster/UI/WBP_ClearResultsWidget`
- `Content/Monster/UI/WBP_BossBattleHUD`
- `Content/GameCore/BP_PlayerController`

## 사용 방법

1. 보스 Ability Set에 사망 Ability를 등록하고 사망 몽타주를 설정한다.
2. RequestBossEncounterEvent의 EncounterTarget에 보스를 지정한다.
3. 사망 후 연출과 BP_GameClear를 PostEvents에 연결한다.
4. 클리어 UI를 EventManager.OnGameClear에 연결한다.
5. 결과를 표시할 상호작용에 BP_ShowClearResults를 연결한다.
6. PlayerController의 ClearResultsWidgetClass에 WBP_ClearResultsWidget을 지정한다.
7. 위젯의 통계 TextBlock과 복귀 버튼을 C++ BindWidget 이름에 맞춰 구성한다.
8. 대기방을 거치지 않고 시작하는 경우 기본 복귀 맵을 설정한다.

## 테스트 및 확인 항목

개별 거리·확률·피해량·쿨다운 수치는 DA, BP 및 몽타주 설정에서 관리한다. 이 문서의 확인 항목은 실행 검증 체크리스트이며 완료 결과를 의미하지 않는다. Blueprint 내부 참조와 ST의 실제 전이는 에디터에서 확인한다.

아래 항목은 실행 검증 체크리스트이며, 이 문서 작성 시 실행 결과를 확인하지 않았다. Blueprint 내부 연결과 PostEvents의 실제 순서는 에디터에서 확인해야 한다.

- [ ] 보스 체력 0 이하에서 사망 Ability 실행 확인
- [ ] 사망 시작·완료 및 PostEvents 중복 실행 방지 확인
- [ ] 사망 시 공격·전환 Ability 취소 확인
- [ ] 사망 몽타주 누락·중단 시 사망 완료 확인
- [ ] 마지막 타격과 처치가 최종 통계에 포함되는지 확인
- [ ] 연출 Delay가 클리어 시간에 포함되지 않는지 확인
- [ ] 서버와 클라이언트의 클리어 UI 표시 확인
- [ ] 클리어 이후 게임오버 차단 확인
- [ ] 클리어 이전 결과 표시 차단 및 위젯 중복 생성 방지 확인
- [ ] 통계 복제가 늦게 도착해도 결과 갱신 확인
- [ ] 결과 화면에서 이동·시점 입력 차단 확인
- [ ] 복귀 준비 인원 복제 및 중복 클릭 방지 확인
- [ ] 준비 중 퇴장 시 남은 인원 기준 복귀 판정 확인
- [ ] 전원 준비 후 파티 단위 대기방 이동 확인
- [ ] 복귀 후 이름·영웅·입장 순서 유지 및 런 초기화 확인
- [ ] 복귀 후 다음 게임 시작 및 새 참가자 접속 확인

## 후속 작업

- 보스 사망부터 대기방 복귀까지 멀티플레이 통합 검증
- Blueprint의 PostEvents 및 결과 표시 상호작용 연결 확인
- 세션 종료·갱신 실패 시 안내 및 재시도 처리 검토
- 늦게 접속한 플레이어의 클리어 UI 복원 정책 검토
