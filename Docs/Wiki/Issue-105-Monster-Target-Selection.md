# #105 몬스터 타깃 선정 재정립

## Issue

- 정리 번호: 105 (포트폴리오 문서 정리용 임의 번호이며, 원본 GitHub 이슈 번호가 아닙니다.)
- 기능명: 멀티플레이 환경 몬스터 타깃 선정 재정립
- 담당 영역: AI / Perception / StateTree
- 선행 이슈: #31 AI Controller 및 State Tree 기본 구조 구현, #47 서버 통합 및 Replication 기본 구조 구현

## 배경

타깃이 고정되는 지점은 두 곳이다.

| 위치 | 동작 |
|---|---|
| `AMonsterAIController::RequestEncounter` | 배틀 섹터를 밟은 인스티게이터 한 명을 섹터의 전 몬스터가 타깃으로 잡는다 |
| `AMonsterAIController::OnPerceptionUpdated` | 시야 감지 상태가 바뀔 때만 재평가한다. 기준은 거리뿐이다 |

재평가가 퍼셉션 이벤트에 묶여 있어, 타깃이 계속 보이는 한 갱신 기회가 오지 않는다. 그 결과 다음 두 상황에서 타깃이 풀리지 않았다.

- 플레이어가 사망해도 시체가 계속 보이므로 콜백이 오지 않는다. `bHasTarget = IsValid(TargetActor)`는 `State.Dead`를 보지 않는다
- `GA_SniperSkill_Camo`가 붙이는 `State.Skill.Camo.Stealth`를 AI가 전혀 읽지 않아, 은신한 스나이퍼가 계속 타깃으로 남는다

## 목적

몬스터가 정해진 주기와 정해진 사건에 따라 타깃을 다시 계산하고, 계산 결과가 기준을 넘을 때만 교체한다.

### 지향

| 상황 | 기대 동작 |
|---|---|
| 4인이 함께 섹터 진입 | 인카운터를 유발한 플레이어가 초기 어그로를 가진다 |
| 앞장선 플레이어에게 몬스터가 몰림 | **허용한다.** 먼저 들어간 것은 어그로를 끌겠다는 선택이다. 동시 공격 수만 어택 토큰이 제한한다 |
| 타깃이 사망·은신 | 즉시 다른 유효 후보로 전환 |
| 타깃이 시야에서 사라짐 (벽 뒤, 문틀 등) | 눈에 보이는 다른 유효 후보가 있으면 즉시 그쪽으로 전환한다. 없으면 `SightMemoryDuration` 동안 기억 후보로 유지해 추격을 잇고, 시간이 지나면 `LostTarget`으로 넘어간다 |
| 다른 플레이어가 공격 | 최근 피격 반응과 실제 적용 피해량 Threat로 어그로가 옮겨갈 수 있다 |
| 등 뒤에서 공격받음 | 시야 밖이어도 후보로 인정해 돌아본다 |
| 두 플레이어가 비슷한 조건 | 혼잡도로 동점만 가른다 (분산 강제 아님) |

### 지양

막아야 하는 것은 **고정(fixation)** 이지 **집중(concentration)** 이 아니다. 둘은 다른 문제다.

| | 뜻 | 판단 |
|---|---|---|
| 고정 | 상황이 변해도 타깃을 갱신하지 않음 | 버그 — 막는다 |
| 집중 | 여러 몬스터가 한 명을 노림 | 설계 — 플레이어 선택의 결과일 수 있다 |

고정을 막는 장치는 셋이다.

1. **주기 재평가** — 상황이 변하면 항상 다시 계산한다
2. **임계값 감쇠** — 같은 타깃을 오래 유지할수록 전환 문턱이 낮아진다
3. **유지 시간 상한** — 상한 도달 시 현재 타깃을 후보에서 제외하고 강제 재선정한다

3번이 있어 유효 후보가 2명 이상이면 고정이 수학적으로 불가능하다.

집중은 막지 않는다. 대신 어택 토큰으로 **동시 공격 수**를 제한해, 어그로를 끈 대가는 치르되 즉사하지 않게 한다.

### 유의

모든 입력은 서버 상태에서 결정적으로 산출하며 난수를 쓰지 않는다. 같은 상황이면 같은 타깃이 나온다. 코옵 게임 중에는 확률 + 불운 방지로 분산시키는 방식도 있으나, "확실한 계산" 요건에 맞지 않아 채택하지 않았다.

## 계산 정의

### 1단계 · 후보 필터

하나라도 불만족이면 후보에서 제외한다.

```text
Character.Player 보유           AND
State.Dead 미보유               AND
State.Skill.Camo.Stealth 미보유  AND
( 현재 시야 감지 중  OR  DamagedTargetMemoryDuration 안에 나를 때림 )
```

#### 시선 기억

후보를 모을 때는 두 층으로 나눈다.

1. **보이는 후보**: 현재 시야 감지 + 피격 예외 + 전원 추적(`bTrackAllPlayers`). 하나라도 있으면 이 후보들만으로 셀렉터가 고른다.
2. **기억 후보**: 보이는 후보가 하나도 없을 때만 쓴다. 퍼셉션이 기억하고 있는(`GetKnownPerceivedActors`) 액터 중 `SightMemoryDuration` 안에 본 대상이다.

기억 후보도 위 후보 필터(`Character.Player`, `State.Dead`, `Stealth`)를 똑같이 거친다. 그래서 죽었거나 은신한 대상은 기억에서도 빠지며, 시체가 계속 보여서 타깃이 풀리지 않던 고정 문제는 되살아나지 않는다.

기억 후보는 눈앞에 보이는 다른 플레이어보다 앞서지 못한다. 후보가 하나뿐일 때만 작동하므로, 벽 뒤로 돌아간 상대를 잠깐 더 쫓는 용도로만 쓰인다. 기억이 만료되면 다음 주기 재평가(0.5초 이내)에서 후보가 비어 타깃이 해제된다.

타깃을 유지하는 것과 그 타깃에게 어떻게 이동하는지는 별개다. 지상 몹은 `MoveToActor`가 네비 경로로 문을 돌아가고, 비행 몹은 직선이 막혔을 때 이동 컴포넌트가 스스로 막힘을 판단해 네비 경로를 따라간다.

### 2단계 · 점수

각 항을 [0,1]로 정규화한 뒤 가중합한다.

```text
D(P) = clamp(1 − Dist(P) / MaxRange, 0, 1)                         // 거리
R(P) = (최근 T_Recent(3초) 내 나를 공격했는가) ? 1 : 0                // 최근 피격
T(P) = clamp(DecayedThreat(P) / DamageThreatCap, 0, 1)             // 피해 Threat
C(P) = clamp(P를 노리는 몬스터 수 / CrowdSaturationCount, 0, 1)     // 혼잡도

Score(P) = W_Dist(1.0)·D(P) + W_Recent(0.6)·R(P) + W_Threat(1.0)·T(P) − W_Crowd(0.15)·C(P)

C(P)의 분모는 전체 몬스터 수가 아니라 CrowdSaturationCount(3) 고정이다.
전체 수로 나누면 방 크기와 등록 순서에 따라 같은 상황의 값이 달라진다.
```

### 피해 Threat

피격 횟수는 사용하지 않는다. 연사 무기, 산탄의 다단 히트, 지속 피해가 불공정하게 높은 어그로를 만들기 때문이다. 서버가 실제 HP에 반영한 `AppliedDamage`만 사용한다.

```text
피격 시:
Threat(P) = min(DamageThreatCap,
                DecayedThreat(P) + AppliedDamage / 몬스터 MaxHealth)

조회 시:
DecayedThreat(P) = Threat(P) × 0.5 ^ (경과 시간 / DamageThreatHalfLife)
```

- `RecentDamage`는 방금 공격한 대상에 즉시 반응하는 이진 보너스다.
- `DamageThreat`는 실제로 얼마나 기여했는가를 반영하는 누적값이다.
- Threat는 공격을 멈추면 자연 감쇠하며, 영구 누적되지 않는다.
- 치명타·약점·방어력 등을 거쳐 실제로 HP가 감소한 값만 누적하므로 무기별 발사 횟수로 인한 왜곡이 없다.

곱셈이 아니라 가중합을 쓰는 이유는 유틸리티 AI(IAUS)에 정리된 함정 때문이다. 고려요소를 곱으로 합치면 항이 늘수록 점수가 0으로 수렴한다.

### 3단계 · 전환 판정

```text
t = 현재 타깃을 유지한 연속 시간(초)

Threshold(t) = max(1.0,  K0 − (t − T_Grace) / T_Decay × (K0 − 1.0))
               K0 = 1.25,  T_Grace = 3초,  T_Decay = 7초

전환 조건:  Score(최고 후보) > Score(현재 타깃) × Threshold(t)
강제 전환:  t ≥ MaxTargetHoldTime(15초) 이고 유효 후보 ≥ 2명
            → 현재 타깃을 제외하고 최고점 재선정
```

`K0 = 1.25`는 MMO 관행에서 가져왔다. WoW는 근접 110% / 원거리 130%를 쓴다. 기존 `TargetSwitchMargin = 0.8`도 같은 개념이며, 거리 대신 점수에 적용하도록 옮긴 것이다.

### 4단계 · 재평가 시점

| 종류 | 조건 |
|---|---|
| 주기 | `TargetEvaluationInterval`(0.5초)마다. 몬스터별 오프셋은 `GetUniqueID() % 100`으로 결정적 분산 |
| 즉시 | 시야 감지 상태 변화 (`OnPerceptionUpdated`) |
| 즉시 | 현재 타깃이 아닌 상대에게 피격 (`NotifyDamagedBy`) |
| 즉시 | `RequestEncounter` 수신 (초기 타깃 결정) |

## 주요 구조

```text
AMonsterAIController
  ├ EvaluateTarget()                  ← 선정의 유일한 진입점
  │    ├ GatherTargetCandidates()     ← 시야·피격 예외·전원 추적 + 후보 필터, 없으면 시선 기억
  │    └ SetCurrentTarget(actor, 사유) ← 교체 + 유지 시간 초기화 + 로그
  ├ 공격자별 DamageThreat 저장·감쇠                    ← 실제 적용 피해량 / MaxHealth
  ├ UMonsterTargetSelector (Instanced)                 ← 선정 정책
  │    ├ UWeightedTargetSelector   (위 계산식, 기본)
  │    └ UNearestTargetSelector    (보스·특수몹용)
  └ UMonsterCombatDirectorSubsystem : UWorldSubsystem
       ├ 몬스터→타깃 배정 레지스트리   (C(P) 산출)
       └ 어택 토큰 풀                (동시 공격 수 제한)
```

- 컨트롤러는 정책을 모른다. 트리거가 오면 셀렉터에 묻고 `TargetActor`에 캐싱만 한다
- 몹 종류별로 셀렉터만 교체하면 성격이 달라진다. GW2도 몹마다 가중치가 다르다
- 읽는 쪽은 바뀌지 않는다. 어빌리티·StateTree·이동은 계속 `GetPerceivedTarget()`만 쓴다
- 전부 서버 전용(AIController는 서버에만 존재)이라 복제 설계가 불필요하다

## 설정값

| 프로퍼티 | 위치 | 기본값 | 의미 |
|---|---|---|---|
| `TargetEvaluationInterval` | `AMonsterAIController` | 0.5 | 주기 재평가 간격(초) |
| `MaxTargetHoldTime` | 〃 | 15.0 | 유지 시간 상한(초). 0이면 강제 전환 없음 |
| `DamagedTargetMemoryDuration` | 〃 | 3.0 | 시야 밖 공격자를 후보로 인정하는 시간(초). 시야각을 180(전방위)으로 열면 등 뒤 예외는 사실상 필요 없어진다 |
| `SightMemoryDuration` | 〃 | 5.0 | 시선이 끊긴 뒤에도 타깃으로 유지하는 시간(초). 보이는 후보가 없을 때만 쓰며, 퍼셉션 `MaxAge`로도 쓰인다. 0이면 시선이 끊기는 즉시 놓친다 |
| `SightRadius` | 〃 | 1000 | 처음 발견하는 거리 |
| `LoseSightRadius` | 〃 | 2500 | 한 번 본 상대를 계속 인지하는 거리. 네비메시가 없는 곳까지 쫓지 않도록 스폰 활성화 거리 안쪽으로 둔다 |
| `PeripheralVisionAngle` | 〃 | 180 | 정면 기준 좌우 편각. 180이면 전방위 |
| `TargetSelector` | 〃 | `UWeightedTargetSelector` | 선정 정책. BP에서 교체 가능 |
| `MaxScoringRange` | `UWeightedTargetSelector` | 3000 | 거리 정규화 기준 |
| `DistanceWeight` | 〃 | 1.0 | 거리 가중치 |
| `RecentDamageWeight` | 〃 | 0.6 | 최근 피격 가중치 |
| `DamageThreatWeight` | 〃 | 1.0 | 감쇠된 피해 Threat 가중치 |
| `CrowdWeight` | 〃 | 0.15 | 혼잡도 감점 가중치 (동점 처리용) |
| `CrowdSaturationCount` | 〃 | 3 | 이만큼 몰리면 혼잡도 최대 |
| `RecentDamageWindow` | 〃 | 3.0 | 최근 피격으로 인정하는 시간(초) |
| `DamageThreatHalfLife` | `AMonsterAIController` | 10.0 | Threat가 절반으로 감소하는 시간(초) |
| `DamageThreatCap` | 〃 | 1.0 | 공격자 한 명이 보관할 수 있는 정규화 전 Threat 최대치 |
| `SwitchScoreRatio` | 〃 | 1.25 | 전환 문턱 시작값 |
| `SwitchGraceTime` | 〃 | 3.0 | 문턱 유지 시간(초) |
| `SwitchDecayTime` | 〃 | 7.0 | 문턱이 1.0까지 내려가는 시간(초) |
| `SwitchDistanceRatio` | `UNearestTargetSelector` | 0.8 | 거리 기반 전환 여유값 |
| `MaxAttackTokensPerTarget` | `UMonsterCombatDirectorSubsystem` | 2 | 한 대상을 동시에 칠 수 있는 몬스터 수 |
| `bRequiresAttackToken` | `FSTT_ActivateAbilityByTag` | false | 켜면 토큰을 얻어야 어빌리티 발동 |

## 도입 순서

| 단계 | 내용 | 상태 |
|---|---|---|
| 1 | 주기 재평가 + 즉시 트리거 + 후보 필터(사망·은신) | 완료 |
| 2 | 빙의 대상 조건을 태그로만 한정 | 완료 |
| 3 | 셀렉터 클래스 분리 | 완료 |
| 4 | 점수식 + 전환 임계값 감쇠 | 완료 |
| 5 | 디렉터 서브시스템 + 분산 감점 + 어택 토큰 | 완료 (ST 상태별 `bRequiresAttackToken` 체크 필요) |
| 6 | 실제 적용 피해량 기반 Threat 누적·감쇠 | 완료 |

어택 토큰은 기본이 꺼짐이다. 공격 계열 State의 `Activate Ability By Tag` 태스크에서 `bRequiresAttackToken`을 켜야 동작하며, 이동·회피·등장 상태에서는 꺼둔다. 가중치와 토큰 상한은 4인 플레이로 조정한다.

## 확인 방법

전환이 일어날 때마다 사유와 유지 시간이 로그로 남는다.

```text
[MonsterTarget] Switch | Controller=... Pawn=... Prev=... New=... Reason=Closer HoldTime=4.2
[MonsterTarget] Score | New=...(D=0.72 R=1.00 T=0.35 C=0.00) Cur=...(D=0.80 R=0.00 T=0.05 C=0.00) Threshold=1.25 Hold=2.0
```

| Reason | 의미 |
|---|---|
| `Encounter` | 배틀 섹터 진입 시 인스티게이터를 폴백으로 지정 |
| `NoValidTarget` | 타깃이 없거나 후보 조건을 잃어 새로 선정 |
| `NoCandidate` | 유효 후보가 없어 타깃 해제 |
| `Score` | 새 후보 점수가 전환 문턱을 넘음 |
| `MaxHoldTime` | 유지 시간 상한 도달로 강제 전환 |

검증 항목

- 플레이어 사망 시 `NoValidTarget` 또는 `NoCandidate`가 즉시 찍히는가
- 스나이퍼 은신 시 타깃이 해제되는가
- 같은 타깃을 15초 이상 유지하는 구간이 없는가 (후보 2명 이상일 때)
- 4인 진입 시 몬스터가 한 명에게 몰리지 않는가
- 같은 횟수라도 실제 적용 피해량이 큰 플레이어가 더 높은 Threat를 얻는가
- 공격을 중단한 플레이어의 Threat가 `DamageThreatHalfLife`마다 절반으로 감쇠하는가
- 벽 뒤로 돌아간 플레이어를 `SightMemoryDuration` 동안 계속 쫓고, 만료 뒤 `NoCandidate`로 타깃이 해제되는가
- 벽 뒤 기억 대상보다 눈앞에 보이는 다른 플레이어가 우선되는가
- 기억 중인 플레이어가 죽거나 은신하면 기억 후보에서도 즉시 빠지는가

## 참고 자료

- [Hate (video games) — Wikipedia](https://en.wikipedia.org/wiki/Hate_(video_games))
- [Aggro — Guild Wars 2 Wiki](https://wiki.guildwars2.com/wiki/Aggro)
- [WoW Classic Threat Guide & Reference Table](https://www.warcrafttavern.com/wow-classic/guides/threat-guide-reference-table/)
- [Infinite Axis Utility System — Considerations](https://uintel-go.utilityworlds.com/Documentation/UtilityIntelligence/Considerations/)
- [Ask a Game Dev — 논보스 적 AI와 어택 토큰](https://askagamedev.tumblr.com/post/620010728353595392/when-designing-non-boss-enemy-ai-ie-ai-for)
- [Target Selection for AI Companions in FPS Games (FDG 2014)](https://www.cs.mcgill.ca/~jtremb59/Papers/fdg2014_paper_32.pdf)
