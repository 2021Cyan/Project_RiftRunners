# #29 무기 가챠 천장 시스템 구현

## Issue

- Issue 번호: #29
- 기능명: 무기 가챠 천장 시스템 구현
- 담당 영역: Weapon Generation / PlayerState / Replication
- 관련 의존성: #28 무기 등급 및 가챠 확률 구현
- 연관 이슈: #27 PCG 무기 파츠 시스템 구현, #43 휴식/상점 방 구현

## 목적

상점에서 무기를 반복 생성해도 일정 횟수 안에 Epic 이상과 Legendary를 획득하도록 보장한다.

천장 상태는 플레이어별로 `ABasePlayerState`에 저장한다. 따라서 한 플레이어의 상점 이용이 다른 플레이어의 천장에 영향을 주지 않으며, 적 드랍 결과도 상점 천장 카운터를 바꾸지 않는다.

## 천장 규칙

| 항목 | 기본값 | 동작 |
|---|---:|---|
| Epic 기본 확률 | 5.1% | Legendary 실패 뒤 Epic 판정 |
| Epic Hard Pity | 10회 | 10번째 상점 생성에서 Epic 이상 보장 |
| Legendary 기본 확률 | 0.6% | 매 상점 생성마다 우선 판정 |
| Legendary Soft Pity 시작 | 74회 | 74번째부터 Legendary 확률 증가 |
| Soft Pity 증가량 | 6%p | 매 회 절대 확률 증가 |
| Legendary Hard Pity | 90회 | 90번째 상점 생성에서 Legendary 보장 |

Legendary가 생성되면 Legendary와 Epic 카운터를 모두 0으로 초기화한다. Epic이 생성되면 Epic 카운터만 0으로 초기화하고 Legendary 카운터는 계속 누적한다. 이 규칙으로 10회 Epic 이상 보장과 90회 Legendary 보장을 동시에 유지한다.

## 처리 흐름

```text
상점 상호작용 (서버)
        ↓
현재 PlayerState 카운터 + 1 계산
        ↓
Legendary Hard/Soft/Base 판정
        ↓ 실패
Epic Hard/Base 판정
        ↓ 실패
Common · Uncommon 가중치 판정
        ↓
무기 생성과 개인 픽업 Spawn 성공
        ↓
CommitGeneration → RecordGeneratedWeapon
        ↓
등급에 따라 천장 카운터 갱신 및 OwnerOnly 복제
```

생성 실패 시 `CommitGeneration()`을 호출하지 않는다. 장착 가능한 몸체가 없거나 Pickup Spawn에 실패한 경우, 플레이어는 골드와 천장 횟수를 소비하지 않는다.

## 상태 보관과 복제

`ABasePlayerState`는 다음 값을 가진다.

| 필드 | 용도 |
|---|---|
| `ShopPullsSinceLegendary` | 마지막 Legendary 이후 상점 생성 횟수 |
| `ShopPullsSinceEpic` | 마지막 Epic 이상 이후 상점 생성 횟수 |
| `GeneratedWeaponHistory` | 최근 100개 생성 이력 |
| 누적 등급·카테고리 카운터 | 통계 및 UI 확장용 |

천장 상태와 생성 이력은 `COND_OwnerOnly`로 복제한다. 다른 플레이어에게 개인 가챠 이력과 천장 수치를 전송하지 않는다. `CopyProperties()`에도 포함되어 있어 심리스 트래블 뒤에도 유지된다.

## 구현 파일

- `Source/TeamProject_Fin/GameCore/Weapon/Data/WeaponTypes.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Generation/WeaponGenerationSubsystem.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Generation/WeaponGenerationSubsystem.cpp`
- `Source/TeamProject_Fin/GameCore/BasePlayerState.h`
- `Source/TeamProject_Fin/GameCore/BasePlayerState.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/World/GunObject.cpp`

## 사용 방법

서버에서 `UWeaponGenerationSubsystem::SetShopPitySettings()`로 `FWeaponPitySettings`를 교체하면 다음 생성부터 설정이 적용된다. 테스트에는 `SetRandomSeed()`로 난수 시드를 고정해 경계 회차를 재현할 수 있다.

적 드랍은 이 시스템을 사용하지 않는다. 적 드랍은 GunObject Blueprint의 별도 가중치만 사용한다.

## 테스트 및 확인 내용

- [x] 10번째 상점 생성에서 Epic 이상 보장
- [x] 90번째 상점 생성에서 Legendary 보장
- [x] Epic 이상 및 Legendary 결과에 따라 카운터가 올바르게 초기화됨
- [x] 적 드랍 생성은 상점 천장 카운터에 영향을 주지 않음
- [x] 플레이어별 천장 상태를 OwnerOnly로 복제하고 심리스 트래블에 이월함

## 남은 작업 및 확장 방향

- 상점 UI에서 현재 천장 진행도와 보장 조건 표시
- 밸런싱을 위한 DataAsset 연동
