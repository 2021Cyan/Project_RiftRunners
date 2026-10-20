# Project RiftRunners

Unreal Engine 5.8 기반 3인칭 협동 슈터 팀 프로젝트의 포트폴리오용 소스 저장소입니다.

> 제가 최초 작성한 파일 중 담당 기능에 필요한 코드를 [`Workspace/CYW`](./Workspace/CYW)에 정리했습니다.

[원본 프로젝트](https://github.com/2021Cyan/TeamProject3)

## 담당 기능

- [무기 시스템](./Workspace/CYW/Weapon): 발사·장전·반동, 투사체, 무기 모디파이어와 스마트 트래킹 UI
- [GAS 세팅](./Workspace/CYW/GAS): ASC 초기화, AttributeSet, GameplayTag와 AbilitySet
- [총기 PCG](./Workspace/CYW/GunPCG): 파츠·등급·천장 기반 생성, 상점·상자·드랍 무기 장착
- [공통 코드](./Workspace/CYW/Common): 위 기능에서 직접 참조하는 본인 작성 타기팅·피드백·전투 이벤트 유틸

## 팀원 소개

- 2021Cyan: GAS, Weapon, 무기 보상/가챠
- seok0317: Player, Environment
- horimihorimi: AI, Enemy, Boss
- dalbi090211: Class, Skill, Map, UI

## 기술 스택

- C++ / Unreal Engine 5.8 / Gameplay Ability System
- Git

## 참고

- 코드 열람용 저장소로, **단독 실행·빌드·패키징이 불가능합니다**. 에셋과 다른 팀원의 의존 코드는 포함하지 않았습니다.
- 파일 최초 작성자를 기준으로 선별했으며, 원본 소스 본문·include·저작권 표기를 그대로 보존했습니다. 팀원의 후속 수정과 Unreal 생성 틀이 포함될 수 있으므로 모든 줄의 단독 저작을 의미하지 않습니다.
