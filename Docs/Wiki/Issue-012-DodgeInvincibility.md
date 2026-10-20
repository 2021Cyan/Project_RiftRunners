# Issue #12 구르기 무적 판정(Invincibility) 프레임 연동

## Issue
- **Issue 번호**: #12
- **기능명**: 구르기 회피 동작 간 무적 프레임(I-Frame) 구현 및 데미지 시스템 연동 (C++ 및 Blueprint 통합)
- **담당자: seok0317
- **담당 영역**: Ability System / Animation Montage / Blueprint / C++ Damage Core
- **관련 의존성**: #11 구르기 모션 및 어빌리티 구현, #13 공통 데미지 파이프라인

## 구현 목적
플레이어가 구르기 액션을 수행할 때 특정 구간(체공 시간) 동안 모든 즉발성 피해를 무시하는 무적 프레임(I-Frame)을 제공한다.
- **C++ 파트 (`ExecCalc_Damage`)**: 피격 시 데미지 연산기가 `State.Invincible` 태그를 인지하여 데미지를 0으로 무효화하되, 지속 데미지(DoT)는 기획적 예외로써 무적을 관통하도록 설계한다.
- **블루프린트 파트 (몽타주 및 GE)**: 실제 게임플레이에서 캐릭터가 구르는 '정확한 찰나의 순간'에만 해당 태그가 부여되도록 애니메이션 데이터를 세팅한다. 애니메이션 몽타주의 노티파이 스테이트(Anim Notify State)를 활용하여 시각적 모션과 시스템적 무적 판정을 1프레임의 오차 없이 완벽하게 동기화한다.

## 주요 구조
```text
 AM_Roll (구르기 애니메이션 몽타주)
  └─ AnimNotifyState_ApplyGameplayEffect (구르기 체공 구간 한정 배치)
      └─ GE_Invincible_Dodge (임시 무적 효과 부여)
          └─ 타깃 ASC에 State.Invincible 태그 활성화

 UExecCalc_Damage (C++ 데미지 연산기)
  ├─ 1. Target의 태그 검사 (HasMatchingGameplayTag)
  │   └─ State.Invincible 태그 보유 확인
  ├─ 2. 공격 타입 검사 (Damage.Type.DoT 여부)
  │   ├─ 즉발 데미지 (총알/폭발) -> 데미지 무효화 (Return 0)
  │   └─ 도트 데미지 (독/화상) -> 무적 무시하고 정상 스탯 삭감
  └─ 3. 최종 데미지 산출 후 어트리뷰트 적용
```

## 구현 파일
- **C++**:
  - `Source/TeamProject_Fin/GameCore/Tags/GameplayTagSystem.h/cpp`
  - `Source/TeamProject_Fin/GameCore/Ability/ExecCalc_Damage.cpp`
- **Blueprint/Asset**:
  - `AM_Roll` (구르기 애니메이션 몽타주)
  - `GE_Invincible_Dodge` (무적 태그 부여용 데이터 에셋)

## 주요 구현 내용 및 시스템 역할

### 1. 전역 상태 태그 및 DoT 엣지 케이스 처리 (C++ 코어)
방어력 공식을 전담하는 `UExecCalc_Damage` 내부에서 무적 처리를 최상단 우선순위로 판별한다.
- **즉발 공격 차단**: 데미지 연산이 시작될 때 타겟의 ASC를 검사하여 `State.Invincible` 태그가 발견되면 연산을 즉시 중단하고 데미지를 0으로 처리한다.
- **기획적 예외 처리 (무적 관통)**: 단, 유입된 공격의 페이로드(또는 GE 태그)가 `Damage.Type.DoT` 속성을 가지고 있다면 무적 태그를 강제로 무시하고 방어력 삭감 공식으로 진입시킨다. 이는 "구르고 있다고 해서 몸에 붙은 불이나 맹독이 사라지지 않는다"는 액션 게임의 근본적인 기획 룰을 보장한다.

### 2. 무적 상태 부여 전용 Gameplay Effect 세팅 (Blueprint)
오직 무적 태그(`State.Invincible`)를 캐릭터에게 부여하기 위한 목적의 데이터 전용 이펙트(`GE_Invincible_Dodge`)를 생성했다.
- **경과시간 정책 (Duration Policy)**: `Infinite` (무한)으로 설정했다. 이 이펙트의 실질적인 지속 시간은 애니메이션 몽타주의 노티파이 스테이트(Notify State) 구간에 의해 전적으로 통제되기 때문이다. 노티파이가 끝나는 순간 이펙트도 강제로 제거된다.
- **부여되는 태그 (Target Tags - Granted to Actor)**: 타깃 액터에 `State.Invincible` 태그를 결합하여, 이 이펙트가 유지되는 동안 C++의 데미지 연산기가 즉발 데미지를 0으로 무효화하도록 만든다.

### 3. 몽타주 노티파이를 통한 프레임 단위 정밀 제어 (Blueprint)
구르기 애니메이션 몽타주(`AM_Roll`)의 모든 방향 섹션(Forward, Left, Right, ForwardLeft, ForwardRight)에 대해 정밀한 무적 프레임을 세팅했다.
- **노티파이 스테이트 배치**: 노티파이 트랙에 `AnimNotifyState_ApplyGameplayEffect`를 추가하고, 적용할 이펙트로 앞서 만든 `GE_Invincible_Dodge`를 할당했다.
- **구간(Window) 설정**: 각 방향별 모션을 프레임 단위로 확인하여, 캐릭터가 바닥을 박차고 도약하여 몸을 둥글게 마는 순간 노티파이가 시작되도록 조율했다. 이후 등이나 어깨가 바닥에 닿고 다시 발이 땅에 닿아 자세를 추스르기 직전에 노티파이가 끝나도록 바(Bar)의 길이를 늘려 맞췄다.
- **시각적 일치**: 플레이어가 느끼는 시각적 공중 체공 시간과 실제 시스템의 무적 시간이 100% 일치하여 억울한 피격 판정(불합리함)을 방지한다.

## 테스트 또는 확인 내용
- **노티파이 동기화 테스트**: 에디터 내 애니메이션 프리뷰 창에서 타임라인을 스크러빙(Scrubbing)하며, 노티파이가 활성화되는 구간이 실제 캐릭터가 땅에서 떨어져 있는 체공 구간과 일치하는지 육안으로 검수 완료.
- **인게임 I-Frame 검증**: PIE 플레이 중 적의 투사체가 날아올 때 타이밍을 맞춰 구르기를 시전. 캐릭터의 발이 떨어지는 순간 `State.Invincible` 태그가 켜지며 투사체 데미지가 무시되고, 몽타주 종료 전 발이 땅에 닿는 순간 태그가 즉시 사라짐을 `showdebug abilitysystem` 콘솔 커맨드를 통해 확인 완료.
- **DoT 엣지 케이스 검증**: 화상 디버프(`GE_DotDamage_Base`)가 걸린 상태에서 구르기를 연타하더라도, 무적 판정과 무관하게 틱당 도트 데미지가 정상적으로 들어옴을 확인 완료.
- **Result**: Succeeded

## 남은 작업 또는 확장 방향
- 피격 시 단순히 데미지만 0이 되는 것이 아니라, 무적으로 공격을 흘려냈음을 시각적/청각적으로 피드백해주는 '회피 이펙트(잔상 처리)'나 '저스트 회피 효과음' 등을 추가하여 타격감을 향상시킬 수 있다.
