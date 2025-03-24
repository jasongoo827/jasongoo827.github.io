---
title: AfterLife Engine
tags: [Game Engine, Graphics]
style: fill
color: "#ffcc00"
description: Vulkan 기반으로 직접 개발한 3D 게임 엔진 ALEngine을 소개합니다.
---

블로그의 가장 첫 포스트로 내가 만든 게임엔진을 소개하려고 한다. 프로그래밍을 시작한 이후로 가장 정성을 들여 만든 프로그램이기에, 포스트도 내가 구현한 기능별로 상세하게 기록할 예정이다. 다 만들고 나서 아쉬운 점도 많았지만, 그래도 게임 엔진에 관해 많이 알게 되어 얻어가는 점도 많았다고 생각한다. 이제 AfterLife Engine에 대한 소개를 하겠다. 

---

## AfterLife Engine (2024/10 ~ 2025/03)
ALEngine 프로젝트는 Vulkan API를 활용해 개발한 게임 엔진이다. 42Seoul의 VeryRealEngine 프로젝트를 수행하기 위해 개발됐다. ALEngine은 최신 그래픽 기술을 활용한 3D 렌더링, 물리 시뮬레이션, 애니메이션 시스템, 스크립팅을 포함하고 있으며, 직관적인 에디터(Editor) 를 통해 쉽게 조작할 수 있다. 다음은 엔진의 데모 영상이다.

- [Renderer](https://www.youtube.com/watch?v=cwIg2w3mOJ0)
- [Physics](https://www.youtube.com/watch?v=oJnp3A-QEsE)
- [Animation](https://youtu.be/M6dmDZbce60)

![Editor Preview](https://github.com/Very-Real-Engine/ALEngine/blob/main/docs/images/Editor.png?raw=true)


---
## 프로젝트에서 나의 역할
ALEngine의 전체적인 구조를 만들고, 렌더링 엔진, 물리 엔진을 이식했다. 그리고 엔진의 기능을 시험해 볼 수 있는 에디터를 만들고 C#으로 게임 코드를 작성할 수 있게 스크립팅을 구현했다. 팀장의 역할도 수행했다. 처음부터 팀장은 아니었지만, 점점 개발 과정에서 회의를 주도적으로 진행하게 되었고 프로젝트 일정 관리, 문제 상황 공유 등 개발 외적인 일도 많이 하게 되면서 팀장 역할을 수행하게 됐다.

---

## 앞으로 올릴 Post들
내가 구현한 부분들에 관해 상세하게 시리즈별로 올릴 예정이다. 물리 엔진 broadphase -> 엔진 구조 -> Editor -> Scripting 순서 대로 진행할 것이다. 또 렌더링 엔진이나 물리 엔진에 관한 부분은 추후에 간략하게 정리할 예정이다. 
