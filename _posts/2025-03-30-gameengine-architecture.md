---
title: Game Engine Architecture - Intro
tags: [Game Engine, Physics]
style: fill
color: dark
description: ALEngine Architecture 에 대한 소개글.
---

## Game Engine Architecture
게임 엔진은 **게임**을 만들기 위해 필요한 핵심 기능들을 제공하는 소프트웨어 프레임워크이다. 게임 엔진과 같이 거대한 프로그램을 개발하려면, 개발 전에 전체적인 프로그램의 흐름과 구조에 대해 설계하는 것이 필수이다. 먼저 일반적인 게임 엔진의 구조가 어떻게 생겼는지 살펴보자.

![Engine Architecture](https://hightalestudios.com/wordpress/wp-content/uploads/2017/03/jaKUP.png)

위 이미지는 게임 엔진에 일반적으로 들어가는 모듈들이 어떤 구성요소로 이루어져 있는지 간단하게 명시해놓았다. 핵심적인 요소들만 골라서 설명하겠다.


### Rendering Engine
**렌더링 엔진**은 게임 세계를 시각적으로 표현하는 핵심 모듈이다. 2D/3D 오브젝트를 카메라 관점에서 계산하여 화면에 출력하며, 다양한 시각 효과와 최적화 기술을 포함한다.

### Physics Engine
**물리 엔진(Physics Engine)** 은 게임 내 오브젝트들이 현실적인 움직임과 상호작용을 하도록 시뮬레이션하는 시스템이다. 중력, 충돌, 마찰, 반사 등 실제 물리 법칙을 기반으로 한 동작을 처리하여 몰입감을 높인다.

### Animation
**Animation**은 게임 속 캐릭터나 오브젝트의 움직임을 시간에 따라 부드럽게 표현하는 시스템이다. 스켈레탈 애니메이션, 키프레임, 블렌딩 등을 통해 생동감 있는 동작을 구현한다.

### Scene Optimizations
Scene graph, Culling, LOD를 통해 렌더링을 최적화해 CPU, GPU 리소스를 줄인다. 

### Profiling & Debugging
엔진 성능을 높이고 개발을 원활하게 할 수 있게 만드는 도구. 메모리 누수, CPU 소모량, fps 등 다양한 수치를 검사할 수 있게 도와준다. 

### Gameplay Foundation
**Gameplay Foundation**은 게임의 규칙, 상호작용, 캐릭터 동작, 상태 전이 등 실제 플레이를 구성하는 핵심 로직과 시스템을 의미한다. Scripting, Event System등을 활용해 엔진의 low level까지 알 필요없이 게임을 개발할 수 있다.


이렇게 소개한 모듈 말고도 정말 많은 것들이 있다. 그리고 개발자의 필요에 다라 다양한 기능들이 추가될 수도 있는 것이 게임 엔진이다. 

## ALEngine Architecture
VeryRealEngine 과제의 요구사항은 다음과 같다.

### **Engine Basics**

- **Scene Loading**: 파일(예: JSON 형식)을 통해 오브젝트 위치 및 정보를 로드.
- **3D Rendering**: .OBJ 파일을 불러와 텍스처 및 비텍스처 모델 렌더링.
- **Physics Simulation**: 충돌 감지, 중력, 마찰 등을 포함한 물리 시스템.
- **Scene Hierarchy**: 부모-자식 관계를 포함한 오브젝트 관리.
- **Optimization**: **Frustum Culling**, **Occlusion Culling** 사용, 60FPS 이상 유지.
- **재질 및 텍스처**: 다양한 재질과 텍스처 지원.

### **Engine composition**

- **Lighting & Shadow Rendering**: 다중 광원 지원 및 동적 그림자 렌더링.
- **Scene Load System**: JSON 또는 유사 형식 지원, 조명 및 오브젝트 속성 관리.

과제의 요구사항만 충족시킨다면 게임 엔진이 아닌 Scene Loader가 될 것 같았다. 내가 만들고 싶었던 것은 Scripting을 통해 게임의 코드를 작성하고, ui를 통해 Scene을 편집할 수 있고, 각종 편의 기능들을 활용해 게임을 쉽게 만들 수 있는 정말 **게임 엔진**을 만드는 것이었다. 그래서 추가적으로 구현한 기능은 **Scripting**, **Editor**, **Scene Serializer**이다. 이 기능들이 추가된 것을 도식화하면 다음과 같다. 

![ALEngine Architecture](https://github.com/Very-Real-Engine/ALEngine/raw/main/docs/images/EngineArchitecture.png)


ALEngine의 구조가 어떻게 구성되어 있는지 설명을 해보았다. 이렇게만 봤을 때는 엔진의 흐름이 어떻게 돌아가는지 알 수가 없다. 하나의 포스트에 모든 설명을 담기에는 너무 글이 길어질 것 같기 때문에 시리즈로 여러 포스트를 올리려고 한다. 엔진 루프에 대해서도 자세히 설명할 예정이다. 하지만 그 전에 프로젝트 환경설정은 어떻게 했는지에 대한 포스트를 작성하려고 한다. 환경 설정을 하며 많은 것을 공부할 수 있었기 때문이다.

ALEngine Architecture 시리즈는 다음과 같은 순서로 진행할 예정이다. 

Intro -> Project Settings -> Engine Loop -> Event System -> ECS(Entity Component System)