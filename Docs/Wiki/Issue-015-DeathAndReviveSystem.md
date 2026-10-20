# Issue #15 사망 및 부활 시스템 구현

## 1. Issue
- **Issue 번호**: #15
- **기능명**: 사망 및 부활 시스템 통합 파이프라인 및 상태 시각화 구현
- **담당자**: seok0317
- **담당 영역**: Ability System / Interaction / Blueprint / UI / Material

## 2. 목적
GAS(Gameplay Ability System)를 기반으로 플레이어의 체력이 0이 되었을 때의 사망 처리부터, 팀원의 협동을 통한 장판(Zone) 부활, 그리고 부활 시 부여되는 무적 버프 및 데미지/피격 UI 연출까지의 전체 생명주기를 완벽하게 구현한다.

## 3. 주요 구조
```text
[어트리뷰트 체력 0 도달] 
  ↓ (Event.Death)
UGA_Death (사망 몽타주 재생, 입력 차단, 장판 스폰)
  ↓ 스폰
AReviveZoneActor (부활 장판)
 ├─ TSubclassOf<UReviveGaugeWidget> ReviveGaugeClass
 └─ 서버 주도 오버랩 판정 및 15초 게이지 계산
  ↓ (15초 도달 -> Event.Revive)
UGA_Revive (사망 해제, 체력 회복, 무적 버프 부여)
  ↓ (GameplayCue 발동)
AGC_Invincible (무적 이펙트를 골반 소켓에 강제 부착)

[상시 UI 및 연출]
WBP_GamePlayBase (전투 HUD)
 └─ UDamageOverlayWidget (ASC 체력 바인딩, 체력 저하 및 피격 시 외곽 비네트 연출)
```

## 4. 관련 구현 파일 (C++)
- `Source/TeamProject_Fin/GameCore/Ability/GA_Death.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/GA_Revive.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Ability/Cue/GC_Invincible.h/.cpp`
- `Source/TeamProject_Fin/GameCore/Interact/ReviveZoneActor.h/.cpp`
- `Source/TeamProject_Fin/GameCore/UI/ReviveGaugeWidget.h`
- `Source/TeamProject_Fin/GameCore/UI/DamageOverlayWidget.h/.cpp`

---

## 5. C++ 코어 시스템 세부 구현 내용

### 5.1 사망 흐름 (Death Flow)
- **사망 감지**: `ExecCalc_Damage` 등에서 체력이 0 이하로 떨어지면 `Event.Death` 태그를 날려 `UGA_Death` 어빌리티를 강제 실행한다.
- **입력 제어**: `APlayerCharacter::Move` 함수 내부에서 캐릭터가 `State.Dead` 태그를 가지고 있으면 이동을 즉시 차단(Return)한다. 단, 마우스 화면 회전(Look)은 허용하여 누워있는 동안 아군의 전투를 관전할 수 있게 한다.
- **장판 생성**: 사망 몽타주가 시작됨과 동시에 서버 권한으로 사망한 플레이어 위치에 `AReviveZoneActor`를 스폰한다.

### 5.2 협동 부활 (Revive Zone)
- **서버 주도 검증**: 살아있는 팀원이 반경 300의 캡슐 영역에 오버랩되면 `ActiveRescuers` 배열에 추가된다. 서버의 `UpdateLoop`에서 매 틱(0.05초)마다 유효한 구조자가 있는지 검증하고 게이지를 올린다.
- **이탈 시 유예 기간(Grace Period)**: 구조자가 밖으로 나가도 2초간 게이지가 유지되며, 이후 1초당 3초 분량씩 게이지가 빠르게 깎인다(Decaying).
- **UI 최적화 및 캐싱 (ESC 버그 방지)**: 
  - 부활 게이지(`UReviveGaugeWidget`)는 메뉴 창(`UDefaultWidgetLayer`)이 아닌 일반 `UUserWidget`으로 구현한다.
  - 잦은 렉을 방지하기 위해 장판 액터가 단 1회만 `CreateWidget`으로 생성하고, 구조자가 오갈 때 `Visibility(Visible/Collapsed)`만 토글한다.
  - `Destroy()` 파괴 시퀀스 도중 좀비 위젯이 스폰되는 엔진 버그를 막기 위해 `IsActorBeingDestroyed()` 방어 로직을 삽입했다.

### 5.3 부활 성공 및 무적 큐 (Gameplay Cue)
- **부활 처리**: 게이지가 15초(Max)에 도달하면 장판이 `Event.Revive` 페이로드와 함께 `UGA_Revive`를 실행하고 자신을 파괴한다.
- **State 갱신**: `UGA_Revive`에서 `State.Dead`를 지우고 `GE_Revive`를 적용하여 `State.Invincible` 태그와 `GameplayCue.Character.Invincible` 큐를 발동시킨다.
- **무적 이펙트 강제 부착 (`AGC_Invincible`)**:
  - `AGameplayCueNotify_Actor`를 상속받는 C++ 클래스로 구현.
  - 엔진이 마음대로 캡슐 컴포넌트(발밑)에 이펙트를 부착해버리는 현상을 막기 위해 `bAutoAttachToOwner = false;`로 설정.
  - `WhileActive` 단계에서 Skeletal Mesh의 특정 뼈(`TargetBoneName` = pelvis)를 찾아 강제로 스냅 부착시켜 이펙트가 캐릭터 몸통 중심을 따라다니도록 보장한다.

### 5.4 체력 및 피격 데미지 오버레이 (UI 연출)
- **상시 구조 변경**: 데스 스크린을 폐기하고, 전투 HUD 내부에 부품으로 들어가는 `UDamageOverlayWidget`으로 재설계.
- **실시간 델리게이트 바인딩**: `NativeConstruct` 시점 로컬 플레이어의 `AbilitySystemComponent`에 접근해 `Health` 어트리뷰트 변경 델리게이트를 바인딩하여 Tick 의존도를 제거.
- **다이나믹 머티리얼 연출**: 
  - 체력이 50% 밑으로 떨어질 때부터 외곽에 피가 맺히기 시작하며, 0% 도달 시 시야 전체가 붉게 물든다.
  - 피격 시 델리게이트가 구형 값(Old)과 신형 값(New)을 비교해, 감소했을 때만 `HitFlashAlpha`를 치솟게 하고 C++ `NativeTick`으로 부드럽게 감쇠시킨다.

---

## 6. 블루프린트 및 데이터 에셋 세팅

### 6.1 다이나믹 피격/체력 머티리얼 구성
- `HealthRatio`와 `HitFlashAlpha` 스칼라 파라미터를 기반으로 동작.
- **비선형 외곽 마스크**: `RadialGradientExponential`에 `OneMinus`와 `Power(지수 3~5)` 노드를 거쳐 중앙 시야는 깨끗하게 유지하고 가장자리 쪽에만 마스크를 날카롭게 형성함.

### 6.2 부활 게이지 채도(HSV) 변화 연출
- 부활 게이지 바 비율(0.0~1.0)을 `HSV to RGB` 노드의 채도(Saturation) 값으로 활용하여 하얀색에서 빨간색으로 게이지 색상 실시간 변경.

### 6.3 메타사운드 동기화
- 부활 진행률(VisualProgress)에 비례하여 사운드의 피치(Pitch)와 템포를 점진적으로 상승시키고, 부활 성공 시 클라이맥스 트리거를 전송.

---

## 7. 사용 방법 (에디터 세팅)

1. **부활 장판 세팅**: `BP_ReviveZone` 액터의 디테일 패널 -> `Revive Settings | UI` 카테고리에서 `ReviveGaugeClass`에 `WBP_ReviveGauge` 할당.
2. **큐 에디터 갱신**: 창(Window) -> `GameplayCue Editor`를 열어 `BP_GC_Invincible`을 GAS 시스템이 인식하도록 등록 갱신.
3. **HUD 배치**: 상시 HUD(`WBP_GamePlayBase`) 캔버스에 `WBP_DamageOverlay`를 추가하고, 앵커를 **전체 채우기(Fill)** 로 설정.

---

## 8. 시스템 테스트 및 해결한 트러블슈팅 (Bug Fixes)

### 테스트 내역
- [x] 체력이 0이 되었을 때 즉시 사망 상태로 전환되고 장판이 서버에 스폰되는가?
- [x] 구조자가 장판에 들어갔다 나올 때 부활 게이지가 서버-클라 간 정상 동기화되며, UI 생성 렉이 없는가?
- [x] 부활 성공 시 무적 이펙트가 발밑(캡슐)이 아닌 골반(Pelvis) 위치에 정확히 달라붙는가?
- [x] 데미지를 입거나 체력이 50% 밑으로 떨어졌을 때 화면 가장자리에 피격 오버레이가 정상적으로 노출되는가?

### 해결한 주요 트러블슈팅
1. **파괴 시퀀스 좀비 위젯 버그 (Destruction Sequence Zombie Widget)**
   - **증상**: 부활에 성공하여 호스트 서버의 장판이 파괴될 때, 부활 게이지 위젯이 화면에 영원히 지워지지 않고 남는 현상.
   - **해결**: `Destroy()` 호출 시 동기적으로 `EndPlay`가 돌아 위젯을 지우지만, 함수 반환 후 남은 `UpdateLoop`가 실행되며 죽어가는 액터가 새 위젯을 다시 스폰하는 엔진 동작 확인. `UpdateLoop` 하단에 `if (IsActorBeingDestroyed()) return;` 방어 코드를 추가하여 완벽히 해결.
2. **부활 장판 최초 오버랩 무시 버그 (BeginOverlap Issue)**
   - **해결**: 장판 생성 직후(`InitializeZone`) 캡슐에 이미 겹쳐 있는 액터들을 `GetOverlappingActors`로 긁어와 수동으로 `OnOverlapBegin`을 강제 호출해주도록 픽스함.
3. **피격 오버레이 전체 화면 붉어짐 버그 (Material Math Logic)**
   - **해결**: 머티리얼 내에서 마스크 연산이 모두 끝난 최종 단계에 `HitFlashAlpha`를 단순 더하기(Add) 하던 로직을 수정. 베이스 투명도에 먼저 더한 뒤 그라데이션 마스크(Radial Mask)를 곱하도록 순서를 변경하여 테두리 쪽에만 피격 효과가 먹히도록 수학 수식 수정.
4. **무적 이펙트 발밑 스폰 버그 (GameplayCue Auto-Attach)**
   - **해결**: `AGameplayCueNotify_Actor`의 기본 설정인 `bAutoAttachToOwner`를 `false`로 끄고, C++ `WhileActive` 단계에서 스켈레탈 메시의 `pelvis`(골반) 소켓을 찾아 수동으로 스냅 부착(Snap to Target)하도록 강제함.

---

## 9. 남은 작업 또는 확장 방향
- **힐 오버레이 추가**: 향후 회복 틱마다 오버레이 머티리얼에 초록색 플래시 파라미터를 추가 전송하여 피드백 강화.
- **데스 스크린 애니메이션**: 체력 0 도달 시 WBP_DamageOverlay 내에서 "You Died" 텍스트 페이드인 위젯 애니메이션 연동.
