# Pixel_Survivor

Unity와 C#을 활용하여 제작한 **Vampire Survivors 스타일의 2D 탄막 액션 로그라이크 개인 프로젝트**입니다.  
흥행작 Vampire Survivors의 핵심 게임 루프를 분석하고 재구성하며, 자동 공격, 성장 선택지, 아이템/무기 확장 구조, Object Pooling 기반 최적화를 직접 구현했습니다.

## Project Info

| 항목 | 내용 |
|---|---|
| 작업 기간 | 2022.09 ~ 2022.12 |
| 인력 구성 | 1인 개발 |
| 사용 언어 | C# |
| 개발 환경 | Unity |
| 플랫폼 | PC / Android |
| 기여도 | 100% |
| 역할 | 기획, 시스템 설계, 클라이언트 구현, 최적화 |

## Gameplay Video

[플레이 영상 보기](https://youtu.be/MosRU7KqfcE?si=SuZuSTvUVTePqvTj)

## Overview

Pixel_Survivor는 다수의 적을 상대하며 경험치를 획득하고, 레벨업 시 무기 또는 능력치를 선택해 성장하는 구조의 게임입니다.

주요 구현 목표는 다음과 같습니다.

- Vampire Survivors 스타일의 핵심 게임 루프 구현
- 자동 공격 기반 무기 시스템 설계
- 레벨업 선택지 및 성장 시스템 구현
- ScriptableObject 기반 데이터 중심 설계
- Object Pooling을 통한 성능 최적화
- PC / Android 멀티 플랫폼 대응

## Core Features

### 1. OOP 기반 캐릭터 시스템

추상 클래스 `Character`를 기반으로 `Player`와 `Enemy`를 구현했습니다.
공통 속성과 행동은 상위 클래스에서 관리하고, 플레이어와 적의 개별 기능은 하위 클래스에서 확장하는 구조로 설계했습니다.

```text
Character
├── Player
└── Enemy
```

주요 설계 특징:

- `CharacterData`를 ScriptableObject로 분리하여 캐릭터 데이터 관리
- Player / Enemy 공통 로직 재사용
- 기능별 클래스를 분리하여 유지보수성과 확장성 확보

### 2. 무기 및 능력치 시스템

`Weapon` 추상 클래스를 기반으로 다양한 무기 타입을 확장할 수 있도록 설계했습니다.
무기 데이터는 `WeaponData` ScriptableObject로 분리하여, 코드 수정 없이 공격력, 쿨타임, 아이콘, 설명 등을 관리할 수 있도록 구성했습니다.

```text
WeaponData
└── Weapon
    └── WeaponSpawner
```

주요 설계 특징:

- 무기 타입별 데이터 분리
- 무기 생성 및 공격 흐름을 `WeaponSpawner`에서 관리
- 신규 무기 추가 시 데이터와 하위 클래스 확장만으로 대응 가능

### 3. 레벨업 및 성장 시스템

레벨업 시 정해진 개수의 선택 슬롯을 동적으로 생성하고, ScriptableObject 기반 아이템 데이터를 랜덤으로 가져와 UI에 표시했습니다.

아이템 선택 시 `Inventory`의 `Dictionary`에 데이터를 저장하여, 이미 보유한 아이템은 레벨업 처리하고 새로운 아이템은 신규 획득 처리하도록 구현했습니다.

```text
Inventory Dictionary
Key   : Item ID
Value : ItemData
```

구현 요소:

- 동적 레벨업 슬롯 생성
- 랜덤 아이템 선택지 표시
- `Dictionary.ContainsKey()` 기반 보유 여부 확인
- 무기와 악세서리 효과 분리 적용
- ParticleSystem과 Color.Lerp를 활용한 연출

## Technical Problem Solving

### Object Pooling 기반 성능 최적화

#### Problem

몬스터, 투사체, 아이템 드롭처럼 반복적으로 생성되고 삭제되는 GameObject가 많아지면서 `Instantiate()`와 `Destroy()` 호출이 빈번하게 발생했습니다.
이로 인해 CPU 부하와 GC 스파이크가 증가하고, 프레임 드랍이 발생하는 문제가 있었습니다.

#### Cause

Unity에서 GameObject를 반복 생성/삭제하면 메모리 할당과 해제가 지속적으로 발생합니다.
특히 탄막 액션 장르에서는 적 생성, 투사체 발사, 드롭 아이템 생성이 반복되기 때문에 GC 호출 빈도가 높아질 수밖에 없었습니다.

#### Solution

- Object Pooling 구조를 도입하여 객체를 생성/삭제하지 않고 재사용
- 사용이 끝난 객체는 비활성화 후 Pool에 반환
- 재사용 가능한 적, 투사체, 아이템 오브젝트를 Pool 단위로 관리
- ScriptableObject를 활용하여 데이터와 런타임 로직 분리

#### Result

| 항목 | Before | After | 개선 |
|---|---:|---:|---:|
| 프레임 시간 | 127ms | 33ms | 약 4배 개선 |
| GC Allocate Frame | 480MB | 176MB | 약 50% 감소 |

Object Pooling 적용 후 프레임 드랍이 감소했고, 메모리 할당량이 줄어 안정적인 플레이 환경을 만들 수 있었습니다.

## Design Decisions

### 왜 ScriptableObject를 사용했는가?

게임 밸런스 데이터와 런타임 로직을 분리하기 위해 사용했습니다.
캐릭터, 무기, 아이템 데이터를 에셋으로 관리함으로써 코드 수정 없이 데이터를 조정할 수 있고, 신규 콘텐츠 추가도 쉬워졌습니다.

### 왜 상속 구조를 사용했는가?

Player와 Enemy가 공통적으로 사용하는 체력, 공격력, 이동 속도, 피격/사망 처리 등을 `Character` 기반으로 관리하기 위해 사용했습니다.
이를 통해 중복 코드를 줄이고, 캐릭터 타입 확장 시 유지보수성을 높일 수 있었습니다.

### 왜 Object Pooling을 적용했는가?

탄막 액션 장르 특성상 생성과 삭제가 매우 빈번합니다.
따라서 런타임 중 불필요한 메모리 할당을 줄이고 안정적인 프레임을 유지하기 위해 Object Pooling을 적용했습니다.

## What I Learned

- Unity에서 반복적인 `Instantiate()` / `Destroy()` 호출이 성능에 큰 영향을 준다는 점을 체감했습니다.
- ScriptableObject 기반 데이터 설계가 유지보수성과 확장성에 효과적이라는 점을 학습했습니다.
- 상속과 다형성을 활용하면 게임 시스템을 더 구조적으로 확장할 수 있다는 점을 경험했습니다.

## Future Improvements

- Addressables를 활용한 리소스 로딩 구조 개선
- ECS/DOTS 기반 대량 오브젝트 처리 실험
- 모바일 환경에서의 추가 최적화
- 무기 조합 및 진화 시스템 확장

## Tech Stack

- Unity
- C#
- ScriptableObject
- Object Pooling
- OOP

## Portfolio

자세한 기술 정리와 이미지 자료는 Notion 포트폴리오에서 확인할 수 있습니다.
