---
title: Game Engine Architecture - Engine Loop
tags: [Game Engine]
style: border
color: success
description: ALEngine의 자세한 흐름에 관한 글.
---

## ALEngine Loop
AfterLife 엔진의 구성이 어떻게 이루어져 있는지 지지난번 포스트를 통해 설명했었다. 오늘은 엔진의 초기화, 업데이트 루프가 어떻게 진행되는지에 대해 얘기해보려고 한다. 처음에 엔진 공부를 시작할 때, 어떻게 흐름을 구성할지 고민을 정말 많이 했었다. 고민 끝에 내린 결론은 "내가 가장 많이 사용해본 엔진인 Unity의 흐름을 공부하고 그대로 따라해보자!"는 것이었다. 

## Unity의 Update Frame 처리 순서
Unity의 프레임 처리를 크게 나누면 다음과 같다. 
#### 1. 입력 처리
- 키보드, 마우스 등 입력 데이터 처리

#### 2. Time 값 갱신
- Time.deltaTime, Time.time 시간 관련 변수들 갱신

#### 3. FixedUpdate() 호출
- 고정 시간 간격으로 실행
- Rigidbody, Force, Collision 같은 물리 연산 처리

#### 4. Physics 처리
- Rigidbody, Collider 등 계산 실행
- 충돌, 위치 이동 등 보정

#### 5. Update() 호출
- 매 프레임 실행됨
- 입력 처리, 게임 로직, Animation Trigger 등에 사용

#### 6. Animator 처리
- Animation 연산 처리(Transition, Blending, etc)

#### 7. LateUpdate() 호출
- Update 후 실행
- 카메라 따라가기 같은 후처리를 위해 사용

#### 8. Coroutine 처리
- yield return로 규정된 시간이 되면 다시 건너가며 실행

#### 9. Rendering

## ALEngine Update Frame
위의 흐름을 참고해 ALEngine의 흐름을 Edit 상태와 Play 상태로 나누어 구분했다. Play 상태는 Editor의 Play 버튼을 눌렀을 때를 의미하고, Edit 상태는 평소의 상태를 의미한다.


### Edit Loop
![EditLoop](/assets/OnUpdateEdit.png)

#### 카메라 위치, 시점 선택
Scene을 어디서 바라볼지 정하는 과정이다. 영화를 찍을 때 감독이 큐 사인을 내리면 카메라로 장면을 촬영하는 것과 비슷하다고 생각하면 쉽다. 카메라는 평소에는 EditorCamera, Play에는 SceneCamera를 사용해 시점을 다르게 보여준다.

#### 가시성 판단, 최적화
Frustum Culling을 사용해 카메라의 시야 범위 밖에 있는 물체들은 렌더링 시 Draw 호출을 하지 않는다. CPU를 덜 쓸 수 있다는 점에서 아주 중요한 최적화이다. 


### Update Loop
![UpdateLoop](/assets/OnUpdatePlay.png)

#### 게임 로직 처리
오브젝트에 부착된 Script가 실시간으로 적용되게 처리한다. 게임이 어떻게 돌아가는지 실질적으로 정의되어 있는 부분이다. 영화로 치면 각 배우의 대사, 그리고 배우 간 상호작용이라고 생각할 수 있겠다. 

#### 물리 계산 및 결과 업데이트
Rigidbody, Collision에 해당하는 물리 연산 처리를 해준다.

#### 애니메이션 업데이트
State Manager, Blending 등을 통해 애니메이션을 처리한다.

## 고려했던 점

#### 멀티쓰레드
"엔진이 싱글 쓰레드로 돌아가도 문제 없을까?" 라는 생각을 많이 했었다. ALEngine은 렌더링, 물리 연산, 애니메이션, 스크립트 기능이 구현되어 있는 간단한 엔진이다. 그렇기 때문에 이 정도 기능으로 부하가 안 생길거야! 라는 생각으로 멀티 쓰레딩을 구현하지 않았다. 하지만 Scene을 엄청나게 많은 Box Collider로 구성하면 부하가 많이 생긴다! 많은 물리 연산 -> 프레임 간 간격 증가 -> Duration 증가 -> 부정확한 물리 처리 이렇게 악순환이 반복됐었다. 이 악순환을 해소하려면 예전에 Broadphase 포스트의 마지막에서 얘기했던 것처럼 Fixed Duration 처리 + 물리 연산용 쓰레드로 최적화해야 한다.

#### 카메라가 없을 때
Editor Camera는 항상 존재하지만, Scene에 있는 Camera를 없애면 Play를 눌렀을 때 아무것도 안 보여야하지 않을까? Unity를 시험해봤을 때, Scene Camera를 없애면 다음과 같이 나온다.

![NoCam](https://europe1.discourse-cdn.com/unity/original/3X/4/c/4c094899eed4fded2a651c1e50a00089b607ec2e.png)

그래서 ALEngine도 비슷하게 처리했다. 우리 렌더링 팀원 surkim이 손수 제작한 이미지다...

![ALNoCam](/assets/nocam.png)

#### Frustum Culling
Frustum Culling은 카메라에 따라 절두체 공간을 계산하고, Culling을 할지 말지 결정했다.

```
void Renderer::beginScene(Scene *scene, Camera &camera)
{
	camera.setAspectRatio(viewPortSize.x / viewPortSize.y);
	projMatrix = camera.getProjection();
	viewMatirx = camera.getView();

    // 절두체 계산
	scene->frustumCulling(camera.getFrustum());
	drawFrame(scene);

	scene->initFrustumDrawFlag();
}


// Draw 호출
MeshRendererComponent &meshRendererComponent = view.get<MeshRendererComponent>(entity);
if (meshRendererComponent.cullState == ECullState::RENDER)
{
    meshRendererComponent.m_RenderingComponent->draw(drawInfo);
}

```

약간 아쉬웠던 점은 처음부터 Frustum Culling을 고려하고 코드를 설계한 게 아니라, 구현 과정에서 정말 많이 터졌었다. 특히 MeshRendererComponent를 추가하거나 삭제하는 과정에서 동기화가 잘 안되는 문제가 있었다. 이는 Culling을 구현한 seonjo와 함께 MeshRenderer를 추가, 삭제하는 모든 부분에서 Culling에 관한 속성을 어떻게 관리할지 상의 후 해결했었다. 

오늘은 ALEngine의 큰 흐름에 대해 설명해보았다. 당시에 고민했던 점들이 좀 더 드러나는 글이면 좋겠지만, 메모를 해놓은 게 많지 않아 더 자세히 쓸 수 없는 점이 아쉽다. 앞으로 포스트들은 프로젝트를 진행하면서 조금씩 쓰는 게 좋을 것 같다. 그래도 ALEngine에 관해 최대한 자세히 기록하고, 여러 방면으로 고민했던 점들을 떠올리고자 노력해볼 것이다. 다음은 "Event System"으로 돌아오겠다!