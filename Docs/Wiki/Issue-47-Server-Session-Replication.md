# #47 서버 통합 및 Replication 기본 구조 구현

## Issue

- Issue 번호: #47
- 기능명: 서버 통합 및 Replication 기본 구조 구현
- 담당자: dalbi090211
- 담당 영역: GameFramework / Session / Replication
- 관련 Sprint: 4주차

## 목적

Steam 세션을 이용한 매치메이킹, 리슨 서버를 통한 게임 플레이 구현.

해결해야 할 문제는 두 가지였다.

1. **플레이어가 서로를 찾는 방법이 없다.** 접속 주소를 직접 입력하는 것 외에 방을 만들고 목록에서 고르는 경로가 필요하다.
2. **플레이어별 데이터의 소유자가 정해져 있지 않다.** 선택한 영웅과 닉네임은 대기실에서 정해져 게임 맵까지 살아남아야 한다.

이번 단계의 구현 범위는 다음과 같다.

- Steam 세션 기반 방 생성 / 검색 / 참가
- 리슨 서버 개시와 대기실 진입
- 게임 시작 시 전원 동시 이동

창을 띄우고 닫는 UI 레이어 시스템은 [#48 전투 HUD](Technical-Docs-Issue-48-Combat-HUD)에서 다룬다. 여기서는 그 시스템 위에 올라가는 로비 화면들만 다룬다.

## 주요 구조

```text
ABaseGameMode                     클래스 등록 + 접속 처리
 ├─ PlayerControllerClass  → ABasePlayerController
 ├─ PlayerStateClass       → ABasePlayerState
 ├─ GameStateClass         → ABaseGameState
 ├─ PostLogin              → 로드아웃에 표시 이름 채움
 └─ bUseSeamlessTravel     → 트래블 시 참가자 유지

ABasePlayerState                  플레이어별 데이터의 소유자
 └─ FDLPlayerLoadout Loadout      Replicated, OnRep_Loadout

ABaseGameState
 └─ OnRoomPlayersChanged          PlayerArray 변경 / 로드아웃 변경 신호

ALobbyPlayerController            매치메이킹 (ABasePlayerController 상속)
 └─ IOnlineSession                Steam 세션 생성/검색/참가
```

전체 흐름은 다음과 같다.

```text
방 만들기
 └─ CreateGame(RoomName)
     └─ CreateSession (Steam 로비로 광고)
         └─ World->Listen()            맵 이동 없이 리슨 서버 개시
             └─ 대기실 UI 표시

방 참가
 └─ FindGames()  → FindSessions → 방 목록
     └─ JoinGame(Entry) → JoinSession
         └─ GetResolvedConnectString → ClientTravel
             └─ 호스트 월드 진입, BeginPlay 에서 대기실 자동 오픈

게임 시작
 └─ StartGame()  (호스트만)
     └─ StartSession (목록에서 내림)
         └─ ServerTravel(GameplayMapName)  심리스, 전원 동시 이동
```

## 구현 파일

### GameFramework

- `Source/TeamProject_Fin/GameCore/BaseGameMode.h/.cpp`
- `Source/TeamProject_Fin/GameCore/BaseGameState.h/.cpp`
- `Source/TeamProject_Fin/GameCore/BasePlayerState.h/.cpp`
- `Source/TeamProject_Fin/GameCore/BasePlayerController.h/.cpp`

### 매치메이킹

- `Source/TeamProject_Fin/GameCore/Match/LobbyPlayerController.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Match/SessionEntryData.h`
- `Source/TeamProject_Fin/GameCore/Match/SessionEntryWidget.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Match/RoomPlayerEntryWidget.h/.cpp`
- `Source/TeamProject_Fin/GameCore/UI/DLPlayerLoadOut.h`

### 설정

- `TeamProject_Fin.uproject` (OnlineSubsystem, OnlineSubsystemSteam, OnlineSubsystemUtils, SteamSockets)
- `Source/TeamProject_Fin/TeamProject_Fin.Build.cs` (OnlineSubsystem, OnlineSubsystemUtils)
- `Config/DefaultEngine.ini` (Steam, SteamSockets 넷드라이버)
- `Config/DefaultGame.ini` (`MapsToCook`)
- `steam_appid.txt`

## 데이터 소유권

### 플레이어 데이터는 PlayerState

```cpp
USTRUCT(BlueprintType)
struct FDLPlayerLoadout
{
    UPROPERTY(BlueprintReadOnly) FString PlayerName;
    UPROPERTY(BlueprintReadOnly) EDLHero HeroType = EDLHero::Bruiser;
};
```

`ABasePlayerState`가 이 구조체를 `ReplicatedUsing = OnRep_Loadout`으로 보유한다. PlayerState는 접속한 플레이어마다 하나씩 서버가 만들고 전원에게 복제하므로, 대기실에서 고른 영웅이 별도 전송 코드 없이 모두에게 보인다.

기본값을 `None`이 아니라 `Bruiser`로 둔다. 아무것도 고르지 않고 시작하는 경우를 없애기 위함이고, 서버와 클라이언트가 같은 기본값으로 구조체를 만들기 때문에 복제가 도착하기 전에도 표시가 어긋나지 않는다.

이름은 접속 시점에 서버가 한 번 채운다. `AGameModeBase::InitNewPlayer`가 이미 PlayerState의 이름을 채워 놓으므로 그것을 로드아웃으로 옮기기만 하면 된다.

```cpp
void ABaseGameMode::PostLogin(APlayerController* NewPlayer)
{
    Super::PostLogin(NewPlayer);
    if (ABasePlayerState* PS = NewPlayer->GetPlayerState<ABasePlayerState>())
    {
        PS->SetPlayerNickname(PS->GetPlayerName());   // Steam 이면 페르소나 이름
    }
}
```

변경은 서버 권한으로만 한다. 클라이언트에서 직접 쓰면 다음 복제 때 덮어써진다.

```cpp
ALobbyPlayerController::SelectHero(EDLHero)
    └─ Server_SelectHero (Server RPC)
        └─ ABasePlayerState::SetHeroType()
            └─ 복제 → OnRep_Loadout → 전원 UI 갱신
```

이 경로는 **자기 것만 바꿀 수 있다.** RPC 인자에 대상 플레이어가 없고, 서버는 RPC를 보낸 컨트롤러의 PlayerState만 만지기 때문이다. UI에서 따로 막을 필요가 없다.

### 인원 목록은 PlayerArray

대기실 인원 목록을 위한 복제 배열은 두지 않는다. `AGameStateBase::PlayerArray`가 이미 접속자 전원의 PlayerState 목록이고 엔진이 복제한다. 배열을 하나 더 만들면 두 벌이 어긋나는 문제만 생긴다.

`ABaseGameState`는 데이터를 갖지 않고 변경 신호만 낸다.

```cpp
virtual void AddPlayerState(APlayerState*) override;     // → OnRoomPlayersChanged
virtual void RemovePlayerState(APlayerState*) override;  // → OnRoomPlayersChanged
```

로드아웃이 바뀔 때도 `ABasePlayerState::NotifyLoadoutChanged()`가 같은 델리게이트를 울린다. 서버는 `OnRep`이 돌지 않으므로 세터에서 직접 호출한다. 대기실 UI는 이 신호 하나만 구독하면 입장·퇴장·영웅 변경이 모두 처리된다.

### 신호를 놓치는 구간

클라이언트는 접속 직후 GameState가 아직 없다. 그래서 `BindRoomUpdates()`가 `GameStateSetEvent`를 기다렸다가 붙는데, **그 시점에 이미 도착해 있던 PlayerState는 `AddPlayerState`를 지나간 뒤**다. 붙기만 하고 아무 신호도 받지 못하면 목록이 한 번도 그려지지 않는다.

바인딩 직후 한 번 브로드캐스트해서 이 구간을 덮는다.

```cpp
GS->OnRoomPlayersChanged.AddDynamic(this, &ALobbyPlayerController::OnRoomUpdated_Broadcast);
OnRoomUpdated_Broadcast();   // 붙기 전에 도착한 것들을 그리게 한다
```

같은 이유로 `BeginPlay`에서는 **대기실을 먼저 띄우고 그다음에 바인딩**한다. 순서가 반대면 위젯이 만들어지기 전에 신호가 지나간다.

## 세션 (Steam)

### 방 생성

```cpp
FOnlineSessionSettings Settings;
Settings.bIsLANMatch = false;
Settings.NumPublicConnections = MaxPlayers;
Settings.bShouldAdvertise = true;
Settings.bAllowJoinInProgress = true;    // 시작할 때 내린다
Settings.bUsesPresence = true;
Settings.bUseLobbiesIfAvailable = true;  // Steam 로비로 광고
Settings.Set(SEARCH_KEYWORDS, RoomName, ...);
Settings.Set(GameKeyName, GameKey, ...);
```

생성이 끝나면 **맵을 갈아타지 않고** 현재 월드에서 넷 드라이버만 연다.

```cpp
FURL URL;
GetWorld()->Listen(URL);
```

`ServerTravel(맵 + "?listen")`을 쓰면 호스트가 로비 화면에서 다른 맵으로 튕겨 나간다. 대기실을 UI로 구성하려면 호스트가 있던 자리에 그대로 있어야 하므로 `UWorld::Listen()`을 사용한다.

이 때문에 **PIE에서는 방 만들기를 테스트할 수 없다.** PIE 월드에는 이미 넷 드라이버가 있어서 그 위에 다시 리슨을 열면 연결이 끊기고 전원이 기본 맵으로 돌아간다.

### 방 검색과 식별 키

테스트 단계에서는 Steam appid 480(Spacewar)을 쓴다. 등록비 없이 동작하지만 전 세계 개발자가 같은 appid를 공유하므로 남의 테스트 방이 목록에 섞인다. 이를 막기 위해 세션 설정에 식별 키를 광고하고 검색에서 같은 값으로 필터한다.

```cpp
const FName GameKeyName(TEXT("DLGAMEKEY"));
```

서버측 필터가 무시되는 경우가 있어 `OnFindComplete`에서 받은 결과의 키를 한 번 더 확인한다.

키 값 뒤의 버전 문자열은 네트워크 호환이 깨지는 변경을 했을 때 올린다. 구버전 빌드와 방 목록이 자동으로 분리되어 버전이 다른 클라이언트끼리 접속하는 사고를 막는다.

### 참가

```cpp
JoinSession → GetResolvedConnectString → ClientTravel(주소, TRAVEL_Absolute)
```

`SteamSockets` 넷드라이버를 사용하므로 이 주소는 IP가 아니라 SteamID다. Steam 릴레이를 통해 연결되어 호스트의 포트포워딩이 필요 없다.

참가자는 트래블 후 호스트 월드에서 컨트롤러가 새로 만들어진다. 이때만 권한이 없으므로 그것으로 판별해 대기실을 연다. 혼자 로비 맵을 켠 상태는 Standalone 이라 권한이 있어 걸리지 않는다.

```cpp
if (!HasAuthority())
{
    ShowWidget(ELayerName::MatchRoom);
}
```

### 게임 시작

```cpp
UpdateSession(bAllowJoinInProgress = false);  // 로비 입장을 막는다
StartSession(NAME_GameSession);               // 방 목록에서 내린다
GetWorld()->ServerTravel(GameplayMapName);
```


생성 시점에 `bAllowJoinInProgress = false`로 두면 안 된다. 엔진은 로비 멤버가 바뀔 때마다 `FillMembersFromLobbyData()`에서 `bLobbyJoinable = bAllowJoinInProgress && (인원 < 정원)`으로 joinable을 다시 계산해 `SetLobbyJoinable()`을 호출한다. false면 첫 참가자가 들어온 순간 로비가 잠겨서, 정원이 4여도 호스트 포함 2명에서 더 받지 못한다. 그래서 대기실 동안은 열어 두고 시작할 때 `UpdateSession`으로 내린다.

`ABaseGameMode`는 `bUseSeamlessTravel = true`로 둔다. 논-심리스 트래블은 클라이언트 연결을 끊고 새 맵에서 재접속시키는 방식이라 리슨 서버에서 참가자가 그대로 떨어져 나간다. 심리스면 접속을 유지한 채 넘어가고 PlayerController / PlayerState도 살아남아 로드아웃이 게임 맵까지 따라간다.

### 방 목록 데이터

```cpp
UCLASS(BlueprintType)
class USessionEntryData : public UObject
{
    FString RoomName;
    TArray<FDLPlayerLoadout> CurrentPlayers;
    int32 MaxPlayers;
    int32 PingMs;
    int32 SearchResultIndex;   // FOnlineSessionSearchResult는 BP 노출이 안 된다
};
```

검색 결과로는 **몇 명인지만** 알 수 있고 누가 있는지는 오지 않는다. `CurrentPlayers`에는 점유한 자리 수만큼 빈 항목을 넣어 `2 / 4` 표시를 살리고, 실제 인원 정보는 방에 들어간 뒤 GameState에서 읽는다.

`USessionEntryWidget`은 `IUserObjectListEntry`를 구현해 ListView에 그대로 들어간다. `JoinButton`이 있으면 `NativeOnInitialized`에서 클릭을 자동으로 연결하고, 꽉 찬 방이면 버튼을 비활성화한다. 핑에 따라 `PingIcon`의 색이 바뀐다.

## 접속 시점에 대한 결정

OSS의 세션 설정은 **호스트 소유의 키-값 묶음**이다. 클라이언트는 읽을 수만 있고 쓸 수 없으므로, 접속하지 않은 상태에서 자기 닉네임과 영웅을 호스트의 방에 올릴 방법이 없다. 이것이 가능한 것은 멤버별 속성을 지원하는 **로비(Lobby)** 이며, UE 5.8에는 `OnlineServicesSteam`이 없고 `OnlineServicesOSSAdapter`에도 `Sessions`만 있고 `Lobbies`가 없다. Steamworks SDK를 직접 호출하지 않는 한 접근할 수 없다.

따라서 참가 시점에 접속하는 구성을 사용했지만 대기실은 여전히 UI를 사용한다.
때문에 호스트는 맵을 옮기지 않고, 참가자는 호스트가 있는 같은 로비 맵으로 들어오므로 화면상 창이 하나 열릴 뿐이다.

## 설정

### 플러그인

`OnlineSubsystem`, `OnlineSubsystemSteam`, `OnlineSubsystemUtils`, `SteamSockets`

### `Config/DefaultEngine.ini`

```ini
[/Script/Engine.GameEngine]
+NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="/Script/SteamSockets.SteamSocketsNetDriver",DriverClassNameFallback="/Script/OnlineSubsystemUtils.IpNetDriver")

[OnlineSubsystem]
DefaultPlatformService=Steam

[OnlineSubsystemSteam]
bEnabled=true
SteamDevAppId=480
bInitServerOnClient=true
bUseSteamNetworking=false

[/Script/SteamSockets.SteamSocketsNetDriver]
NetConnectionClassName="/Script/SteamSockets.SteamSocketsNetConnection"
```

### `Config/DefaultGame.ini`

이름으로만 여는 맵은 참조가 없어서 자동 쿠킹되지 않는다. 패키징 빌드에 넣으려면 목록에 적어야 한다.

```ini
[/Script/UnrealEd.ProjectPackagingSettings]
+MapsToCook=(FilePath="/Game/TPSDevelop/Map/PCGTestMap")
+MapsToCook=(FilePath="/Game/TPSDevelop/Map/StaticMap/Alloc")
...
```

`DirectoriesToAlwaysCook`(Additional Asset Directories to Cook)은 에셋용이라 **맵에는 먹지 않는다.** 폴더로 지정해도 맵은 빠진다. 섹터 레벨을 추가하면 이 목록에도 넣어야 한다.

### 컨트롤러 설정 (`EditDefaultsOnly`)

| 항목 | 기본값 | 설명 |
|---|---|---|
| `MaxPlayers` | 4 | 방 정원 |
| `GameKey` | `TeamProject3_v1` | 방 목록 필터용 식별자 |
| `LobbyMapName` | `LobbyTestMap` | 나가기 시 돌아올 맵 |
| `GameplayMapName` | `PCGTestMap` | 시작 시 이동할 맵 |

## 사용 방법

### 레벨 구성

1. `ALobbyPlayerController`를 상속한 BP를 만들고 `BaseLayerClass`에 로비 베이스 레이어 WBP를 지정한다.
2. `ABaseGameMode`를 상속한 BP를 만들고 `PlayerControllerClass`에 1번을 지정한다.
3. 로비 맵의 World Settings → GameMode Override에 2번을 지정한다.

`ALobbyPlayerController`를 쓰는 BP가 없으면 로비 화면의 모든 `Cast To LobbyPlayerController`가 실패한다. 캐스트 결과를 변수에 담아 두는 구성이라면 그 변수가 None이 되어 버튼을 눌러도 아무 일도 일어나지 않는다.

### 로비 UI 버튼 연결

모두 `ALobbyPlayerController`의 함수다. `Get Owning Player` → `Cast To LobbyPlayerController`로 접근한다.

| 화면 | 요소 | 연결 |
|---|---|---|
| 방 찾기 | Refresh 버튼 | `FindGames()` |
| 방 찾기 | ListView | `OnSessionListUpdated` 바인딩 → `FoundSessions`를 `Add Item` |
| 방 찾기 | 엔트리 Join 버튼 | 연결 불필요 (C++에서 자동) |
| 방 만들기 | Create 버튼 | `CreateGame(RoomNameBox 텍스트)` |
| 방 만들기 | 결과 처리 | `OnRoomCreated(bSuccess)` 바인딩 |
| 대기실 | 인원 목록 | `OnRoomUpdated` 바인딩 **+ Event Construct** → `GetRoomPlayerEntries()` → ListView `Set List Items` |
| 대기실 | 영웅 선택 | 엔트리의 `HeroCombo` (연결 불필요, C++에서 자동) |
| 대기실 | Start 버튼 | `StartGame()` — `IsHost()`가 false면 비활성 |
| 대기실 | Leave 버튼 | `LeaveRoom()` |

대기실 목록은 `Event Construct`에서도 한 번 그려야 한다. 위젯이 열릴 때 이미 전원이 들어와 있으면 그 뒤로 변경 신호가 오지 않기 때문이다.

`OnRoomJoined(bSuccess)`는 **실패했을 때만** 도달한다. 성공하면 즉시 트래블하므로 콜백이 올 자리가 없다.

## 테스트 및 확인 내용

- [x] 에디터 / Game 타겟 빌드
- [x] 패키징
- [x] 방 생성, 다른 PC에서 방 목록 표시
- [x] 참가 및 대기실 진입, 인원 목록 표시
- [x] 영웅 선택 반영
- [x] 게임 시작 시 전원 이동

Steam 서브시스템은 **PIE에서 신뢰할 수 없다.** 세션 관련 동작 확인은 아래 절차로만 가능하다.

1. Development 로 패키징한다. Shipping 은 로그가 없어 실패 원인을 볼 수 없다.
2. 실행 파일 옆에 `steam_appid.txt`를 두고 내용을 `480`으로 채운다. 패키징에 자동으로 실리지 않으므로 직접 복사한다.
3. **서로 다른 Steam 계정으로 로그인한 두 대의 PC**에 같은 빌드를 넣고 각각 실행한다.
4. 한쪽에서 방을 만들고 다른 쪽에서 목록에 뜨는지, 참가 후 서로의 닉네임과 영웅이 보이는지 확인한다.

세션을 거치지 않는 부분(인원 목록 복제, 로드아웃 복제, 영웅 선택 RPC)은 PIE의 `Play As Listen Server` + 플레이어 2명으로 확인할 수 있다. 두 번째 창이 실제 클라이언트가 된다. 단 이 상태에서 방 만들기를 누르면 위에 적은 넷 드라이버 충돌로 전원이 튕긴다.

## 남은 작업

- 대기실 목록에서 자기 줄을 시각적으로 구분 (`bIsLocal`이 BP에 노출되어 있다)
- 강퇴, 레디 상태

## 확장 방향

```text
매치메이킹
 ├─ 방 비밀번호 / 친구만 초대 (Steam 초대 사용)
 ├─ 레디 상태 및 전원 레디 시 시작
 ├─ 강퇴
 └─ appid 정식 등록 후 GameKey 필터 제거
```
