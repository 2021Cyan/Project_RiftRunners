# #28 무기 등급 및 가챠 확률 구현

## Issue

- Issue 번호: #28
- 기능명: 무기 등급 및 가챠 확률 구현
- 담당 영역: Weapon Generation / Data / Replication
- 관련 의존성: #27 PCG 무기 파츠 시스템 구현
- 연관 이슈: #29 무기 가챠 천장 시스템 구현, #30 무기 드랍 및 장착 교체

## 목적

상점과 적 드랍에서 생성되는 PCG 무기에 등급을 부여하고, 등급에 따라 파츠 레벨과 접두사 효과가 달라지도록 한다.

등급 판정과 생성 결과 확정은 서버만 수행한다. 클라이언트는 서버가 생성해 복제한 `FGeneratedWeaponData`만 사용하므로, 클라이언트별 확률 판정 차이나 조작 가능성이 없다.

## 등급과 생성 규칙

현재 `EGunRarity`는 Common, Uncommon, Epic, Legendary의 네 단계다.

| 등급 | 총 파츠 레벨 | 특수 효과 |
|---|---:|---|
| Common | 3~5 | 없음 |
| Uncommon | 5~7 | 없음 |
| Epic | 7~10 | 없음 |
| Legendary | 7~10 | Modifier DT에서 효과와 접두사 1개 무작위 선택 |

총 파츠 레벨을 먼저 굴린 뒤 총열과 탄창 레벨에 분배한다. 두 레벨은 각각 1~5 범위로 제한한다. 따라서 등급은 희귀도 표기뿐 아니라 최종 스탯 범위와 Legendary 접두사에 직접 영향을 준다.

## 확률 정책

### 상점

상점은 Legendary와 Epic을 순서대로 독립 판정한다.

1. Legendary 판정: 기본 0.6%
2. Legendary가 아니면 Epic 판정: 기본 5.1%
3. 둘 다 아니면 Common : Uncommon = 2 : 1 가중치 판정

Epic 5.1%는 Legendary 판정이 실패한 경우에만 적용된다. 따라서 천장을 제외한 최종 체감 확률은 Legendary 0.6%, Epic 약 5.07%, Common 약 62.89%, Uncommon 약 31.44%다.

### 적 드랍

적 드랍은 상점 확률과 천장을 사용하지 않는다. 적 드랍용 `GunObject` Blueprint의 `EnemyDropRarityWeights`가 개별 확률을 결정한다.

기본 가중치는 Common 60, Uncommon 30, Epic 9, Legendary 1이다. 합계가 100이면 그대로 백분율이며, 가중치이므로 다른 합계도 비율로 처리된다. 모두 0이면 Common으로 안전하게 처리한다.

## 주요 구조

```text
AGunObject::SpawnGeneratedWeapon (서버)
        ↓
UWeaponGenerationSubsystem::GenerateWeapon
        ↓
RollRarity(생성 출처, 플레이어별 천장 상태, 적 드랍 가중치)
        ↓
등급별 파츠 레벨 · RuntimeStats · 접두사 조립
        ↓
FGeneratedWeaponData 복제 및 개인 픽업 생성
```

`GenerateWeapon()`은 해당 플레이어가 장착 가능한 무기 카테고리의 몸체만 후보에 넣는다. 예를 들어 Rifle만 장착 가능한 플레이어는 적 드랍에서도 Rifle 결과만 생성된다.

### Legendary 모디파이어 선택

상점·적 드랍·상자는 공통 생성기에서 프레임의 Modifiers를 비운다. Legendary이면 DT_Modifier의 전체 행에서 동일 확률로 하나 선택하고, 해당 효과 하나와 같은 행의 PrefixName만 적용한다. 다른 등급에는 모디파이어와 접두사를 부여하지 않는다.

DT가 미지정이거나 비어 있으면 추가 모디파이어 없이 생성한다. 접두사가 비어 있는 행도 효과 선택 대상이며, 선택되면 무기 이름에서 접두사를 생략한다. 확률은 효과 종류가 아닌 행 단위이므로 같은 효과를 여러 행에 넣으면 그 효과의 선택 확률이 높아진다.

## 구현 파일

- `Source/TeamProject_Fin/GameCore/Weapon/Data/WeaponTypes.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Generation/WeaponGenerationTypes.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Generation/WeaponGenerationSubsystem.h`
- `Source/TeamProject_Fin/GameCore/Weapon/Generation/WeaponGenerationSubsystem.cpp`
- `Source/TeamProject_Fin/GameCore/Weapon/World/GunObject.h`
- `Source/TeamProject_Fin/GameCore/Weapon/World/GunObject.cpp`

## 사용 방법

- 상점 확률 및 천장 수치는 `FWeaponPitySettings`를 통해 서버에서 변경한다.
- 적 드랍 확률은 몬스터의 `EnemyDropGunObjectClass`로 지정된 GunObject Blueprint의 Class Defaults에서 수정한다.
- Legendary만 시험하려면 적 드랍 가중치를 Common 0, Uncommon 0, Epic 0, Legendary 1로 설정한다.

## 테스트 및 확인 내용

- [x] 서버가 등급, 파츠 레벨, 접두사를 생성하고 결과를 복제함
- [x] 플레이어 장착 가능 카테고리만 생성 후보에 포함됨
- [x] Legendary에 SmartTracking 접두사가 적용됨
- [x] 적 드랍 Blueprint별 가중치가 생성에 전달됨

## 남은 작업 및 확장 방향

- 상점 확률을 데이터 에셋 또는 밸런스 테이블로 이전
- 등급, 파츠 레벨, 접두사를 HUD와 상점 UI에 표시
