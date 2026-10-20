# 기술 문서

각 GitHub Issue별 구현 내용과 사용법을 보관하는 페이지입니다. 파일명은 세 자리 번호로 정렬하며, `105`는 이 저장소에서 부여한 문서 정리용 번호입니다.

새 기술 문서는 [기술 문서 양식](./TechnicalDocument.md)을 기준으로 작성합니다.

## 작성 기준

각 문서는 아래 항목을 기준으로 정리합니다.

- Issue 번호와 기능명
- 구현 목적
- 주요 구현 내용
- 관련 구현
- 사용 방법
- 테스트 또는 확인 내용
- 남은 작업 또는 확장 방향

## 문서 목록

<!-- TECHNICAL_DOCS_START -->
| 번호 | 문서 | 링크 |
|---|---|---|
| `#1` | Unreal Engine 프로젝트 세팅 | [문서 보기](./Issue-001-Unreal-Engine-Project-Setup.md) |
| `#2` | Gameplay Ability System 기본 세팅 | [문서 보기](./Issue-002-GAS-Setup.md) |
| `#3` | AttributeSet 구현 | [문서 보기](./Issue-003-AttributeSet.md) |
| `#4` | GameplayTag 구조 정의 | [문서 보기](./Issue-004-GameplayTag-System.md) |
| `#5` | GameplayEffect 기본 구조 구현 | [문서 보기](./Issue-005-GameplayEffect.md) |
| `#6` | Issue #6 3인칭 캐릭터 이동 구현 | [문서 보기](./Issue-006-Movement.md) |
| `#7` | Issue #7 TPS 카메라 구현 | [문서 보기](./Issue-007-Camera.md) |
| `#8` | Issue #8 조준 시스템 구현 | [문서 보기](./Issue-008-Aiming.md) |
| `#9` | Issue #9 에임 오프셋 구현 | [문서 보기](./Issue-009-AimOffset.md) |
| `#10` | Issue #10 스태미나 시스템 코어 구현 (C++ / Blueprint 통합 편) | [문서 보기](./Issue-010-StaminaSystem.md) |
| `#11` | Issue #11 구르기(회피) 모션 및 어빌리티 구현 | [문서 보기](./Issue-011-DodgeAction.md) |
| `#12` | Issue #12 구르기 무적 판정(Invincibility) 프레임 연동 | [문서 보기](./Issue-012-DodgeInvincibility.md) |
| `#13` | Issue #13 데미지처리 (공통 데미지 파이프라인 및 물리/스탯 연산 분리 구조 통합) | [문서 보기](./Issue-013-DamageSystem.md) |
| `#14` | 피격 반응 및 그로기 시스템 구현 | [문서 보기](./Issue-014-HitReact.md) |
| `#15` | Issue #15 사망 및 부활 시스템 구현 | [문서 보기](./Issue-015-DeathAndReviveSystem.md) |
| `#16` | 공통 무기 베이스 구현 | [문서 보기](./Issue-016-Weapon-Base.md) |
| `#17` | Issue #17 발사 시스템 구현 | [문서 보기](./Issue-017-Weapon-Fire.md) |
| `#18` | Issue #18 장전 시스템 및 사격 상태 제어 개선 | [문서 보기](./Issue-018-Weapon-Reload.md) |
| `#19` | Issue #19 반동 시스템 구현 | [문서 보기](./Issue-019-Weapon-Recoil.md) |
| `#21` | 돌격병 과부하 스킬 구현 | [문서 보기](./Issue-021-Overdrive.md) |
| `#23` | 유탄 투사체 바운스 구현 | [문서 보기](./Issue-023-Grenade-Bounce.md) |
| `#24` | 폭파병 특이점 스킬 구현 | [문서 보기](./Issue-024-Bomber-Mine.md) |
| `#26` | 스나이퍼 적 투시 스킬 구현 | [문서 보기](./Issue-026-Sniper-Camo.md) |
| `#27` | PCG 무기 파츠 시스템 구현 | [문서 보기](./Issue-027-PCG-Weapon-Parts.md) |
| `#28` | 무기 등급 및 가챠 확률 구현 | [문서 보기](./Issue-028-Weapon-Rarity-Gacha-Probability.md) |
| `#29` | 무기 가챠 천장 시스템 구현 | [문서 보기](./Issue-029-Weapon-Gacha-Pity-System.md) |
| `#30` | 무기 드랍 및 장착 교체 구현 | [문서 보기](./Issue-030-Weapon-Drop-Swap.md) |
| `#31` | AI Controller 및 State Tree 기본 구조 구현 | [문서 보기](./Issue-031-AI-Controller-State-Tree.md) |
| `#32` | 근접형 일반 적 구현 | [문서 보기](./Issue-032-Melee-Enemy.md) |
| `#33` | 원거리형 일반 적 구현 | [문서 보기](./Issue-033-Ranged-Enemy.md) |
| `#34` | 엘리트 일반 적 구현 | [문서 보기](./Issue-034-Elite-Enemy.md) |
| `#35` | 적 체력, 피격 및 사망 처리 구현 | [문서 보기](./Issue-035-Enemy-Health-Hit-Death.md) |
| `#36` | 적 처치 보상 드랍 구현 | [문서 보기](./Issue-036-Enemy-Drop-Reward.md) |
| `#37` | 보스 기본 캐릭터 구현 | [문서 보기](./Issue-037-Boss-Base-Character.md) |
| `#38` | 보스 페이즈 전환 시스템 구현 | [문서 보기](./Issue-038-Boss-Phase-Transition.md) |
| `#39` | Alepsis-Taura 보스 공격 패턴 구현 | [문서 보기](./Issue-039-Boss-Attack-Patterns.md) |
| `#40` | 보스 사망 및 게임 클리어 처리 구현 | [문서 보기](./Issue-040-Boss-Death-Game-Clear.md) |
| `#41` | 노드 기반 PCG 맵 생성 구현 | [문서 보기](./Issue-041-PCG-Map-Generation.md) |
| `#42` | 전투 방 구현 | [문서 보기](./Issue-042-Battle-Room.md) |
| `#46` | Issue #46 동적 스노우 디포메이션(Snow Deformation) 시스템 개발 및 최적화 | [문서 보기](./Issue-046-SnowDeformation.md) |
| `#47` | 서버 통합 및 Replication 기본 구조 구현 | [문서 보기](./Issue-047-Server-Session-Replication.md) |
| `#48` | 전투 HUD 구현 | [문서 보기](./Issue-048-Combat-HUD.md) |
| `#74` | 기술문서: 에임오프셋, 사운드 시스템 및 구르기/부활 버그 수정 | [문서 보기](./Issue-074-AimOffset-SoundSystem.md) |
| `#90` | Issue #90 Snow Deformation V2 (눈 변형 시스템 업그레이드) | [문서 보기](./Issue-090-SnowDeformationV2.md) |
| `#94` | Issue #94 혹한 맵 제작 및 머티리얼 색상 변경 이벤트 추가 | [문서 보기](./Issue-094-WinterMapAndMaterialEvent.md) |
| `정리 105` | 몬스터 타깃 선정 재정립 | [문서 보기](./Issue-105-Monster-Target-Selection.md) |
<!-- TECHNICAL_DOCS_END -->
