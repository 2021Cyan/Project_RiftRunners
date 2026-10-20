# Issue #13 데미지처리 (공통 데미지 파이프라인 및 물리/스탯 연산 분리 구조 통합)

## 1. Issue
- **Issue 번호**: #13
- **기능명**: 공통 데미지 파이프라인 구현 및 블루프린트 타겟팅 연동 구조 정립
- **담당자: seok0317
- **담당 영역**: Ability System / Damage System / Blueprint / Physics Settings

## 2. 목적
스킬과 무기가 개별적으로 방어력 및 방관 계산을 수행하는 파편화를 방지하고, 유지보수가 용이한 **공통 데미지 시스템**을 구축한다. 
- **C++ 파트**: SRP(단일 책임 원칙)에 입각하여, C++ 함수(`ApplyGameDamage`)는 오직 **순수 데미지 이펙트 적용 및 스탯 연산(방어력, 무적, 약점 배수)**만을 전담하도록 역할을 한정한다.
- **블루프린트 파트**: 스킬의 타겟팅 형태(박스, 부채꼴, 스피어 등)와 색적(Trace/Overlap) 로직은 블루프린트 환경에 위임하여 기획적 유연성을 극대화하며, 찾아낸 타겟들을 C++ 단일 진입점(`ApplyGameDamage`)으로 넘겨주는 명확한 파이프라인을 확립한다.

## 3. 주요 구조
```text
스킬/무기 (블루프린트)
 ├─ 타겟팅 로직: Overlap/Trace 등으로 타겟 액터 수집 (광역기일 경우 Array Loop 처리)
 ├─ Make DamagePayload: 스킬 기획 데이터(기본 데미지, 방관, 약점 배수)와 HitResult 조립
 └─ UDamageSystemLibrary::ApplyGameDamage (단일 데미지 적용 진입점)
     ├─ Instigator 및 Target의 ASC(Ability System Component) 추출
     ├─ HitResult 분석 및 약점/팀킬 판정에 따른 동적 GameplayTag 부여
     ├─ SetByCaller를 통한 기획 데이터(Damage, ArmorPenetration 등) 전송
     └─ UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf
         └─ UExecCalc_Damage (RPG 스탯 전용 연산기)
             ├─ Target의 Armor, Source의 AttackPower 등 어트리뷰트 캡처
             ├─ Payload로 넘어온 Armor Penetration 적용 (Effective Armor 산출)
             ├─ 무적(Invincible) 태그 검사 및 예외(DoT) 판별
             └─ 방어력 감쇠 공식 적용 후 최종 데미지로 타겟 어트리뷰트 삭감
```

## 4. 관련 구현 파일 (C++)
- `Source/TeamProject_Fin/GameCore/Ability/DamageSystemLibrary.h/cpp`
- `Source/TeamProject_Fin/GameCore/Ability/ExecCalc_Damage.h/cpp`
- `Source/TeamProject_Fin/GameCore/Tags/GameplayTagSystem.h/cpp`
- `Source/TeamProject_Fin/GameCore/Attribute/CharacterAttributeSet.cpp`

---

## 5. C++ 코어 시스템 세부 구현 내용

### 5.1 FDamagePayload 구조체
데미지 연산에 필요한 핵심 기획 데이터만을 컴팩트하게 담아 C++로 전달하는 공용 페이로드 구조체다.
- **기본 스탯 연동**: `BaseDamage`, `WeakpointMultiplier`, `FriendlyFireMultiplier` 등 데미지 산술식에 필요한 기초 계수를 제공한다.
- **방어력 관통(Armor Penetration)**: 무기/스킬 고유의 방관 수치를 지닐 수 있도록 독립 멤버로 구성되어 적의 방어력을 계산 이전에 상쇄시킨다.
- **DoT(지속 데미지) 세팅**: `DoTDuration`과 `DoTPeriod`를 지원하여, 즉발 공격 외에도 화상/독 등 지속 시간에 걸쳐 주기적으로 피해를 주는 데이터 구성을 지원한다.

### 5.2 UDamageSystemLibrary::ApplyGameDamage
모든 데미지 발생원(총기, 검, 수류탄 장판 등)이 반드시 거쳐야 하는 단일 진입점(Entry Point)이다. 
- **컨텍스트 구성**: Source와 Target의 ASC를 확보한 뒤 `FGameplayEffectContextHandle`을 생성하여 공격의 주체(Instigator)와 타격 지점(HitResult)을 이펙트 컨텍스트에 캡슐화한다.
- **동적 태그 주입**: TargetActor와 InstigatorActor의 GAS 태그를 분석하여 팀킬 여부를 판단하고 `Hit.FriendlyFire` 태그를 동적 주입한다. 또한 HitResult의 재질을 분석해 `Hit.Weakpoint` 태그를 주입하여 ExecCalc가 배수 연산을 할 수 있도록 플래그를 세팅한다.
- **데이터 바인딩**: 구조체로 전달받은 수치들을 `SetByCallerMagnitude`를 통해 정확한 GameplayTag 위치에 매핑시켜, 하부 연산기(`UExecCalc_Damage`)가 이를 추출해 계산할 수 있도록 연결고리 역할을 수행한다.

### 5.3 UExecCalc_Damage (커스텀 연산기)
GAS 기반 데미지 연산기(Execution Calculation)로, 모든 수학적 방어력 공식과 스탯 상호작용을 전담한다.
- **방어력 공식 도입**: `EffectiveArmor = Max(Armor - ArmorPenetration, 0.0)` 적용 후, 최신 스탠다드 경감 공식인 `100 / (100 + EffectiveArmor)`을 통과시켜 체력이 아닌 데미지 역전 버그를 원천 차단하는 밸런스 공식을 적용한다.
- **무적기(Invincible) 우선순위 및 엣지 케이스**: 타겟이 `State.Invincible` 태그를 보유 중일 경우 모든 즉발 데미지는 0으로 무효화된다. 단, `Damage.Type.DoT` 태그가 감지될 경우 "지속 피해는 회피의 무적 프레임을 뚫고 데미지를 입힌다"는 기획 룰에 따라 예외적으로 연산을 강행하도록 설계되었다.

---

## 6. 블루프린트 및 데이터 에셋 세팅

### 6.1 커스텀 트레이스 채널 (TraceAttack) 및 피아 식별
모든 스킬과 무기의 타겟 탐색(색적)은 콜리전 채널을 진영별로 나누어 복잡하게 관리하지 않고, 단일 채널과 GAS 태그(Tag)만을 사용하여 처리한다.
- **TraceAttack 단일 채널 사용**: 공격용 LineTrace, SphereTrace, Overlap 등은 무조건 프로젝트 커스텀 채널인 `TraceAttack` 단 하나만 사용하여 궤적 상의 모든 캐릭터(아군, 적군, 중립)를 일단 전부 검출(Hit)한다.
- **GAS Tag 기반 피아 식별**: 검출된 타겟이 아군인지 적군인지는 콜리전 단계에서 거르지 않는다. 타격 이후 페이로드가 `UDamageSystemLibrary::ApplyGameDamage`로 넘어오면, 내부에서 공격자(Instigator)와 피격자(Target)가 보유한 팩션(진영) 태그를 비교하여 피아 식별을 수행한다.
- **장점**: 충돌 매트릭스를 극도로 쾌적하게 유지할 수 있으며, 물리 연산 부하가 줄어든다. 또한 "적군에게 닿으면 데미지, 아군에게 닿으면 관통 또는 힐"과 같이 복잡하고 유연한 기획을 단순히 블루프린트 배열 필터링이나 C++ 태그 로직만으로 쉽게 구현할 수 있다.

### 6.2 물리 표면(Physical Surface) 및 약점 재질 맵핑
약점 부위 피격 시 C++ 연산기가 이를 식별하고 추가 배수를 적용할 수 있도록 엔진 물리 계층을 연동했다.
- **프로젝트 세팅**: 엔진 설정(Physics)의 Physical Surface 항목에서 `SurfaceType3`을 `Weakpoint`로 전역 등록.
- **피지컬 머터리얼 적용**: `PM_Weakpoint`라는 피지컬 머터리얼을 생성해 스켈레탈 메시의 헤드본(Head Bone) 등 약점 부위 콜리전에 직접 할당. 
- **식별 원리**: 무기에서 발사된 LineTrace가 해당 콜리전을 타격하면 HitResult에 Physical Material 데이터가 포함되며, `ApplyGameDamage`가 이를 확인해 자동으로 약점 판정을 내린다. (참고: 재질 정보를 가져올 수 없는 단순 Box/Sphere Overlap의 경우 약점이 발동하지 않으므로, 정밀 타격 시에는 반드시 Trace 기반 타겟팅을 사용해야 함).

### 6.3 데미지 전용 Gameplay Effect (GE) 구성
- **즉발 데미지 (`GE_Damage_Base`)**: 일반적인 총알/타격용 (Instant + `ExecCalc_Damage`).
- **도트 데미지 (`GE_DotDamage_Base`)**: 화상, 독 등 지속 피해용 (Has Duration + 주기별 `ExecCalc_Damage`).

### 6.4 공통 데미지 파이프라인 노드 페이로드 구성
- **Base Damage**: 스킬/무기의 기본 데미지 수치.
- **Armor Penetration**: 방어구 관통력 적용 수치.
- **Weakpoint Multiplier**: 약점 적중 시 증폭시킬 배수 수치 (기본 1.5배).
- **Friendly Fire Multiplier**: 아군 오사 시 감쇠시킬 배수 수치.
- **Damage Effect Class**: 타겟에게 부여할 GE 클래스 할당.


---

## 7. 전투 기획(Blueprint) 파이프라인 가이드

### 7.1 단일 타격 무기 (히트스캔 총기, 근접 무기 등)
1. 공격 적중 시 **Trace (Line/Box)** 노드에서 출력된 정밀한 `Hit Result`를 추출.
2. `Make Damage Payload` 노드에 기획 스탯(데미지, 방관, 약점 배수) 및 `Hit Result` 할당.
3. 대상에게 `Apply Game Damage` 함수 단일 호출.

### 7.2 광역 공격 및 장판 (수류탄, 부채꼴 브레스 등)
1. 시전 위치에서 **`Sphere Overlap Actors`**, **`Box Trace Multi`** 등 기획 의도에 맞는 형태의 충돌 노드로 타겟 배열(Array) 수집.
2. 수집된 액터 배열에 대해 **`For Each Loop`** 실행.
3. 루프 내부에서 각 액터를 대상으로 가상의 `Hit Result`(Make Hit Result 사용)를 구성해 Payload에 결합 후 `Apply Game Damage` 반복 호출.

---

## 8. 시스템 테스트 및 결과
- **약점 연산 검증**: 헤드샷(PM_Weakpoint) 부위 타격 시 Payload의 `WeakpointMultiplier`(예: 1.5)가 C++ ExecCalc로 정상 인입되어 최종 데미지가 증폭되는 현상 디버그 완료.
- **방어력 및 방관 공식 연산**: 타겟 Armor 100, Payload 방관 0 기준으로 감쇠율 50% 적용 확인. 이후 방관 50 입력 시 EffectiveArmor 50 기반 공식으로 데미지 증가 확인 완료.
- **무적 관통(DoT) 테스트**: 구르기(Invincible) 도중 즉발 데미지는 무시하되, `GE_DotDamage_Base`가 부여한 도트 딜은 무적 프레임을 무시하고 지속 삭감됨을 검증 완료.
- **광역기 퍼포먼스**: 블루프린트 오버랩 기반 20기체 동시 피격 루프 처리 시에도 데미지 누락 및 틱드랍 현상 없이 안정적 분산 처리 확인. 으로 등록했다.
- **피지컬 머터리얼 에셋**: `PM_Weakpoint` 에셋을 생성하고 표면 타입을 지정했다. 이를 스켈레탈 메시 특정 뼈 콜리전에 부착하면, Trace를 통해 획득한 HitResult가 이를 감지하여 약점 판정을 내린다. (단, 단순 Overlap은 재질을 가져오지 못하므로 플레이어 피격 등 약점이 필요 없는 경우에만 사용 권장)


---

## 7. 사용 방법 (스킬 및 무기 제작 시)

### 7.1 블루프린트(Blueprint) 환경에서 사용
1. **단일 타격 (총알 등)**: 트레이스(LineTrace) 직후 `Make Damage Payload` 노드를 꺼내 기획 스탯과 타격된 `Hit Result`를 연결하고 `Apply Game Damage` 호출.
2. **광역 타격 (수류탄, 브레스 등)**: `Sphere Overlap Actors`나 `Box Trace Multi` 노드로 적들을 수집한 뒤, `For Each Loop`를 돌면서 개별 액터마다 `Make Hit Result`를 생성해 페이로드에 꽂고 `Apply Game Damage`를 반복 호출.

### 7.2 C++ 환경에서 사용
**[C++ 사용 예시 코드]**

 DamageSystemLibrary.h 내부 그로기 유형의 enum이름
```cpp
UENUM(BlueprintType)
    enum class EDamageGroggyType : uint8
    {
        None            UMETA(DisplayName = "없음 (일반 데미지)"),                                                                                                                                               
        LightGroggy        UMETA(DisplayName = "약한 그로기 (제자리 피격)"),                                                                                                                                     
        HeavyGroggy        UMETA(DisplayName = "강한 그로기 (넉백 피격)")                                                                                                                                        
    };
```

```cpp
FDamagePayload Payload;
Payload.BaseDamage = 15.0f;
Payload.ArmorPenetration = 20.0f;
Payload.HitResult = HitResult; // 트레이스에서 얻은 타격 정보
 Payload.bCausesGroggy = true; // 피격시 그로기가 걸리는 공격인가 아닌가
// Payload.DamageEffectClass는 클래스 내 UPROPERTY 변수로 블루프린트에서 할당받아 사용

UDamageSystemLibrary::ApplyGameDamage(this, GetInstigator(), Payload);
```

---

## 8. 테스트 및 확인 내용
- **약점 배수 검증**: 적 캐릭터의 머리(`PM_Weakpoint` 할당 부위)를 트레이스 타격했을 때 정상 증폭(1.5배) 디버깅 완료.
- **광역 분리 검증**: 블루프린트에서 Overlap 후 Loop를 통해 개별 타격 방식으로 광역 공격을 수행하여 성능 저하 및 다중 피격 이슈 없음 확인 완료.
- **C++ 수식 검증**: 스탠다드 방어력 산술 공식(100/(100+Armor)) 및 도트 딜 무적 관통 기능 정상 작동 검증 완료.
