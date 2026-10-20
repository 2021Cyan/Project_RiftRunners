# #33 원거리형 일반 적 구현

## Issue

- Issue 번호: #33
- 기능명: 원거리형 일반 적 구현
- 담당자: horimihorimi
- 담당 영역: Enemy AI / GAS / Projectile / Animation
- 선행 이슈: #31 AI Controller 및 State Tree 기본 구조 구현, #35 적 체력·피격 및 사망 처리 구현

## 목적

원거리 몬스터의 등장, 추적, 투사체 공격과 특수 행동을 공용 Enemy 프레임워크에 연결한다. 원거리 전용 State Tree를 복제하는 대신 DA_Range_Definition에 전투 데이터를 조립한다.

## 주요 구조

```text
BP_Monster_Range
    ↓
DA_Range_Definition
├─ DA_Range_AbilitySet
├─ DA_Range_Pattern
├─ DA_Range_MovementProfile
└─ 스탯 행 / 표시 이름 / 등장 몽타주
    ↓
ST_Enemy_Common → 패턴 선택·준비·실행·Recovery
    ↓
GA / 몽타주 Notify → 투사체 및 서버 피해 적용
```

## 설계 결정

### 일반 몬스터 공용 흐름

인지와 상태 전환은 공용 컨트롤러·ST 흐름을 사용한다. 패턴의 거리, 체력 조건, 쿨다운, 가중치와 이동 후보는 Pattern Set에서 설정한다. 이동 설정은 Movement Profile에서 관리한다.

### 피해 판정과 연출 분리

투사체 생성·충돌과 피해 적용은 관련 GA 및 투사체 코드가 처리한다. 실제 공격 수치와 Cue 데이터는 에셋에서 설정한다. 에셋이 존재하는 것만으로 모든 특수 행동이 현재 Pattern Set에 등록됐다고 단정하지 않는다.

## 구현 내용

### 등장 및 패턴 구성

Definition의 Emerge는 바닥·벽·천장 기본 등장 몽타주를 제공한다. 스폰 지점의 배치 방식과 개별 오버라이드를 함께 사용한다. 서버가 Ability Set의 GA를 지급하고 Pattern Set을 활성화한다.

### 원거리 공격

BP_GA_Range_RangedAttack과 원거리 공격 몽타주, BP_Range_Projectile 및 Range_Projectile_CueData로 공격을 구성한다. 패턴 실행 태그, 발사 Notify, 투사체 클래스 및 피해 설정을 맞춰 연결한다.

### 회피와 재배치

회피 GA는 공용 상태 흐름에서 실행한다. 순간이동 재배치는 RepositionForAction 이동 후보와 위치 보정 프로필을 통해 요청한다. 이동 후보의 특성 조건 및 GA 등록을 함께 확인한다.

### 자폭

UGA_SelfDestruct는 접근, Detonate 이벤트 대기, 폭발 피해·Cue 및 종료를 처리한다. 폭발은 중복 실행을 방지한다. 타깃 유실 시 접근 목적지를 유지하는 경로와 몽타주 중단 시 종료 경로를 제공한다. 자폭 완료는 캐릭터 사망 시작·완료 및 완료 이벤트에 연결된다.

## 관련 구현

### C++

- `Source/TeamProject_Fin/GameCore/Character/FlyingEnemyCharacter.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Character/CommonEnemyCharacter.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Movement/FlyingMonsterMovementComponent.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_SelfDestruct.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_TeleportReposition.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Projectile/MonsterProjectile.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Combat/MonsterCombatPositioningComponent.h/.cpp`

### Blueprint / Asset

- `Content/Monster/BP/BP_Monster_Range`
- `Content/Monster/AI/Controller/BP_MonsterAIController_Range`
- `Content/Monster/AI/ST/ST_Enemy_Common`
- `Content/Monster/DA/Range/DA_Range_Definition`
- `Content/Monster/DA/Range/DA_Range_AbilitySet`
- `Content/Monster/DA/Range/DA_Range_Pattern`
- `Content/Monster/DA/Range/DA_Range_MovementProfile`
- `Content/Monster/GAS/GA/BP_GA_Range_RangedAttack`
- `Content/Monster/GAS/GA/BP_GA_Emerge_Range`
- `Content/Monster/GAS/GA/BP_GA_Evade_Range`
- `Content/Monster/GAS/GA/BP_GA_TeleportReposition_Range`
- `Content/Monster/GAS/GA/BP_GA_SelfDestruct`
- `Content/Monster/Actor/BP_Range_Projectile`
- `Content/Monster/DA/Range_Projectile_CueData`
- `Content/Monster/DA/Range_SelfDestruct_Data`
- `Content/Monster/Anim/Montage/Range`

## 사용 방법

1. 원거리 BP에 컨트롤러, 이동 컴포넌트 및 DA_Range_Definition을 연결한다.
2. Definition의 Ability Set, Pattern Set과 이동 프로필을 지정한다.
3. 공격 실행 태그, 발사 Notify 및 투사체 설정을 연결한다.
4. 사용할 회피·재배치·자폭 GA를 Ability Set에 등록하고 관련 이벤트 또는 패턴에 연결한다.
5. 실제 Pattern Set 등록 내용과 몽타주 Notify 연결을 에디터에서 확인한다.

## 테스트 및 확인 항목

개별 거리·확률·피해량·쿨다운 수치는 DA, BP 및 몽타주 설정에서 관리한다. 이 문서의 확인 항목은 실행 검증 체크리스트이며 완료 결과를 의미하지 않는다. Blueprint 내부 참조와 ST의 실제 전이는 에디터에서 확인한다.

- [ ] 배치 방식에 맞는 등장 몽타주 확인
- [ ] Pattern Set 조건에 따른 원거리 공격 선택 확인
- [ ] 투사체 발사·충돌·피해의 서버 판정 확인
- [ ] 회피 및 특성 조건에 따른 재배치 확인
- [ ] 자폭 이벤트 중복 방지 및 타깃 유실·중단 처리 확인
- [ ] 사망 후 공격·이동 및 풀 반환 상태 확인

## 후속 작업

- Blueprint 내부 패턴 등록과 Notify 연결 검증
- 멀티플레이 투사체·Cue 동기화 검증
