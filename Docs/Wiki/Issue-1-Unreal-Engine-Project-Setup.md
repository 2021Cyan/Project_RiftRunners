# #1 Unreal Engine 프로젝트 세팅

## Issue

- Issue 번호: #1
- 기능명: Unreal Engine 프로젝트 세팅
- 담당자: 2021Cyan
- 관련 Sprint: 1주차

## 목적

팀 프로젝트의 공통 개발 기반을 만들고, Unreal Engine 5.8에서 같은 프로젝트 구조로 작업할 수 있도록 초기 설정을 구성합니다.

## 구현 내용

- Unreal Engine 5.8 기반 프로젝트 생성
- 기본 Config 파일 추가
- GitHub Issue, PR, README, Wiki 기반 협업 문서 구조 정리
- GAS 사용을 위한 GameplayAbilities 플러그인 활성화

## 관련 구현

- C++: `Source/TeamProject_Fin`
- Blueprint:
- Asset:

## 사용 방법

프로젝트 루트의 `TeamProject_Fin.uproject`를 Unreal Engine 5.8로 열어 작업합니다.

```text
TeamProject_Fin.uproject
```

C++ 모듈은 `TeamProject_Fin` 기준으로 관리합니다.

## 테스트 및 확인 내용

- [x] Unreal Engine 5.8 프로젝트 파일 생성 확인
- [x] `TeamProject_Fin` Runtime 모듈 확인
- [x] GAS 관련 플러그인 활성화 확인
- [x] 관련 Issue 완료 조건 확인

## 남은 작업

- 없음

## 확장 방향

- 프로젝트 진행에 따라 공통 폴더 구조와 Config 설정을 추가 정리합니다.
- 기능별 문서는 Wiki의 기술 문서 페이지에 계속 추가합니다.
