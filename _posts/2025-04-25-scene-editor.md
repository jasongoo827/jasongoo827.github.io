---
title: Scene, Editor
tags: [Game Engine, ImGui, yaml-cpp]
style: border
color: success
description: ALEngine의 Scene system, gui 에 대한 글.
---

## Scene
지난 포스트까지 ALEngine Architecture에 대한 설명을 마쳤다. 오늘 포스트는 Scene Load System와 ImGui를 이용해 만든 Editor에 대한 것이다. 

게임엔진에서 Scene은 일반적으로 하나의 게임 레벨, 맵, 또는 월드를 구성하는 모든 객체와 데이터의 집합을 의미한다. 플레이어가 보는 하나의 "화면 속 세계" 전체를 의미한다고 볼 수 있다. 보통 Scene은 JSON, YAML 형식 등으로 저장되어 있고 엔진은 Scene 파일을 해석해 객체와 그것을 이루고 있는 데이터들을 만들어낸다. 

---

#### Unity의 Scene
Unity의 Scene 파일은 .unity 확장자를 가지며, 해당 Scene에 배치된 모든 게임 오브젝트와 컴포넌트의 정보를 저장하고 있다. 내부 구조는 YAML 또는 바이너리로 이루어져 있다.

```yaml

GameObject:
  m_ObjectHideFlags: 0
  m_CorrespondingSourceObject: {fileID: 0}
  m_PrefabInstance: {fileID: 0}
  m_Name: "Player"
  m_Components:
    - component: {fileID: 1234}
    - component: {fileID: 5678}
  m_Transform: {fileID: 91011}

Transform:
  m_LocalPosition: {x: 0, y: 1, z: 0}
  m_LocalRotation: {x: 0, y: 0, z: 0, w: 1}
  m_LocalScale: {x: 1, y: 1, z: 1}

```
---

### ALEngine의 Scene
ALEngine은 Unity의 Scene 파일처럼 Scene을 구성하기로 했다. 확장자는 .ale를 갖고, 내부 구조는 YAML 형식이다.

Scene의 구조는 간단하다. Scene 이름, Entity들의 정보, 각 Entity를 이루고 있는 Component의 정보들로 이루어져 있다. 

Scene을 바이너리로 관리했다면 용량 감소, 속도 향상, 보안성 등 여러 면에서 좋았겠지만 텍스트로 관리하면, Scene 파일을 직접 편집할 수도 있고, 평가하는 사람이 보기에도 더 편하기 때문에 굳이 바이너리로 만들지는 않았다. 

```yaml

Scene: 3DExmaple
Entities:
  - Entity: 16143175189520029819
    TagComponent:
      Tag: Plane
    TransformComponent:
      Position: [0, -3, 0]
      Rotation: [-1.57079637, 0, 0]
      Scale: [30, 30, 1]
    RelationshipComponent:
      Parent: 0
      Children:
        []
    MeshRendererComponent:
      MeshType: 3
      MatPath: ""
      IsMatChanged: false
  - Entity: 3406970704759287418
    TagComponent:
      Tag: Scene Camera
    TransformComponent:
      Position: [0, 0, 0]
      Rotation: [0, 0, 0]
      Scale: [1, 1, 1]
    RelationshipComponent:
      Parent: 0
      Children:
        []
    CameraComponent:
      Camera:
        PerspectiveFOV: 0.785398185
        PerspectiveNear: 0.00100000005
        PerspectiveFar: 1000
      Primary: true
      FixedAspectRatio: false
  - Entity: 13031684749665374615
    TagComponent:
      Tag: Directional Light
    TransformComponent:
      Position: [0, 0, 0]
      Rotation: [0, 0, 0]
      Scale: [1, 1, 1]
    RelationshipComponent:
      Parent: 0
      Children:
        []
    LightComponent:
      Type: 2
      ShadowMap: 1
      Position: [0, 0, 0]
      Direction: [0.200000003, -1, 0]
      Color: [1, 1, 1]
      Intensity: 1
      InnerCutoff: 0.976296008
      OuterCutoff: 0.953716934
  - Entity: 24453836486531026
    TagComponent:
      Tag: Sphere
    TransformComponent:
      Position: [0, 0, 0]
      Rotation: [0, 0, 0]
      Scale: [1, 1, 1]
    RelationshipComponent:
      Parent: 0
      Children:
        []
    MeshRendererComponent:
      MeshType: 2
      MatPath: ""
      IsMatChanged: false
```

---

## Scene Serialize, Deserialize
Scene의 정보를 효과적으로 저장하고 엔진 상에 띄우려면 적절하게 직렬화, 역직렬화를 해주어야 한다. 직렬화(Serialization)는 데이터를 저장하거나 전송 가능한 형태로 변환하는 과정을 의미한다. 반대로 역직렬화(Deserialization)는 직렬화된 데이터를 다시 원래의 상태로 복원하는 과정이다. 단순하게 직렬화는 저장, 역직렬화는 불러오기 기능이라고 생각해도 된다. 이를 구현하려면 텍스트로 저장된 Scene 파일을 파싱하는 작업이 필요하다. 나는 yaml-cpp 라이브러리를 사용해 .ale 파일을 파싱했다. 

직접 파서를 구현해보는 것도 좋은 경험이었겠지만, 우리 프로젝트는 제한된 기간 내에 완성하는 것이 목표였기 때문에 직접 구현하지는 않았다. yaml-cpp에 내가 필요로 하는 기능만 추가해 사용했다. glm::vec, UUID(커스텀 클래스)을 직렬화, 역직렬화 하기 위해 encode, decode 기능 특수화를 진행했다.

#### 구현

``` cpp
// Serialize 함수
void SceneSerializer::serialize(const std::string &filepath)
{
	YAML::Emitter out;
	out << YAML::BeginMap;
	out << YAML::Key << "Scene" << YAML::Value << "Untitled";
	out << YAML::Key << "Entities" << YAML::Value << YAML::BeginSeq;
	m_Scene->m_Registry.view<entt::entity>().each([&](auto entityID) {
		Entity entity{entityID, m_Scene.get()};
		if (!entity)
		{
			return;
		}
		serializeEntity(out, entity, m_Scene.get());
	});

	out << YAML::EndSeq;
	out << YAML::EndMap;
	std::ofstream fout(filepath);
	fout << out.c_str();
}

static void serializeEntity(YAML::Emitter &out, Entity entity, Scene *scene)
{
	out << YAML::BeginMap;
	out << YAML::Key << "Entity" << YAML::Value << entity.getUUID();

	// KEY, KEY - VALUE
	if (entity.hasComponent<TagComponent>())
	{
		out << YAML::Key << "TagComponent";
		out << YAML::BeginMap;
		auto &tag = entity.getComponent<TagComponent>().m_Tag;
		out << YAML::Key << "Tag" << YAML::Value << tag;
		out << YAML::EndMap; // Tag
	}

	// TransformComponent
	if (entity.hasComponent<TransformComponent>())
	{
		out << YAML::Key << "TransformComponent";
		out << YAML::BeginMap;
		auto &tf = entity.getComponent<TransformComponent>();
		out << YAML::Key << "Position" << YAML::Value << tf.m_Position;
		out << YAML::Key << "Rotation" << YAML::Value << tf.m_Rotation;
		out << YAML::Key << "Scale" << YAML::Value << tf.m_Scale;
		out << YAML::EndMap; // Transform
	}

    // ...

	out << YAML::EndMap; // End Serialize Entity
}

```

```cpp

// Deserialize 함수
bool SceneSerializer::deserialize(const std::string &filepath)
{
	YAML::Node data;

	try
	{
		data = YAML::LoadFile(filepath);
	}
	catch (const std::exception &e)
	{
		AL_CORE_ERROR("Failed to load .ale file '{0}'\n    {1}", filepath, e.what());
		return false;
	}

	// Scene
	if (!data["Scene"])
	{
		return false;
	}

	// Entities
	auto entities = data["Entities"];
	if (entities)
	{
		for (auto entity : entities)
		{
			// IDComponent
			uint64_t uuid = entity["Entity"].as<uint64_t>();

			// TagComponent
			std::string name;
			auto tagComponent = entity["TagComponent"];
			if (tagComponent)
			{
				name = tagComponent["Tag"].as<std::string>();
			}

			AL_CORE_TRACE("Entity deserialized as ID: {0}, Tag: {1}", uuid, name);

			Entity deserializedEntity = m_Scene->createEntityWithUUID(uuid, name);

			// TransformComponent
			auto tfComponent = entity["TransformComponent"];
			if (tfComponent)
			{
				auto &tf = deserializedEntity.getComponent<TransformComponent>();
				tf.m_Position = tfComponent["Position"].as<glm::vec3>();
				tf.m_Rotation = tfComponent["Rotation"].as<glm::vec3>();
				tf.m_Scale = tfComponent["Scale"].as<glm::vec3>();
				tf.m_WorldTransform = tf.getTransform();
			}
			// ...
        }
	}
	return true;
}


namespace YAML
{
// 템플릿 특수화
template <> struct convert<glm::vec3>
{
	static Node encode(const glm::vec3 &rhs)
	{
		Node node;
		node.push_back(rhs.x);
		node.push_back(rhs.y);
		node.push_back(rhs.z);
		node.SetStyle(EmitterStyle::Flow);
		return node;
	}

	static bool decode(const Node &node, glm::vec3 &rhs)
	{
		if (!node.IsSequence() || node.size() != 3)
			return false;

		rhs.x = node[0].as<float>();
		rhs.y = node[1].as<float>();
		rhs.z = node[2].as<float>();
		return true;
	}
};

template <> struct convert<ale::UUID>
{
	static Node encode(const ale::UUID &uuid)
	{
		Node node;
		node.push_back((uint64_t)uuid);
		return node;
	}

	static bool decode(const Node &node, ale::UUID &uuid)
	{
		uuid = node.as<uint64_t>();
		return true;
	}
};
}

```

---

## Editor
실시간으로 scene의 구성 요소를 편집할 수 있게 gui를 만들었다. 구성 요소의 계층 구조를 표현하는 Scene Hierarchy, Entity의 속성을 자세히 보여주는 Inspector, 프로젝트의 리소스를 시각적으로 보여주는 Content Browser Panel을 만들었다. Editor를 개발할 때 주요 목표는 클릭이나 드래그, 드래그 & 드롭 등 손쉬운 동작을 통해 Scene을 편집할 수 있게 하는 것이었다. 또한 렌더링 담당인 surkim과 물리 시뮬레이션 담당인 seonjo가 자신이 만든 기능을 바로 테스트 해볼 수 있게 만드는 것도 중요했다. 

#### Scene Hierarchy
Scene Hierarchy는 게임이나 애플리케이션에서 Scene을 구성하는 Entity 들의 계층적 구조를 의미한다. 부모-자식 관계(parent-child relationship)로 오브젝트들이 연결된다. 한 오브젝트가 이동하거나 변형(transform)되면, 그 자식들도 같이 영향을 받는다. 트리(Tree) 형태로 구성되어, 복잡한 장면을 논리적으로 관리할 수 있게 한다.

Scene Hierarchy를 구현하기 위해 SceneHierarchyPanel이라는 클래스를 만들었고, 해당 클래스가 담당한 기능은 다음과 같다.

#### 1. Scene을 구성하는 Entity 이름 표시
Entity의 tagComponent 정보를 가져와 ImGui를 통해 이름을 표시할 수 있게 했다.

```cpp
void SceneHierarchyPanel::drawEntityNode(Entity entity)
{
	auto &tag = entity.getComponent<TagComponent>().m_Tag;

	ImGuiTreeNodeFlags flags =
		((m_SelectionContext == entity) ? ImGuiTreeNodeFlags_Selected : 0) | ImGuiTreeNodeFlags_OpenOnArrow;
	flags |= ImGuiTreeNodeFlags_SpanAvailWidth; // 선택 영역이 가장자리까지 넓어지게 설정
	bool opened = ImGui::TreeNodeEx((void *)(uint64_t)(uint32_t)entity, flags, tag.c_str());

	// ...
}
```


#### 2. 계층 구조(부모 - 자식 관계) 표현
부모 - 자식 관계를 표현하기 위해 RelationshipComponent를 만들었고, Component에는 부모와 자식의 정보를 보관할 수 있게 했다.

```cpp
/**
 * @struct RelationshipComponent
 * @brief 개체 간의 부모-자식 관계를 정의하는 컴포넌트.
 */
struct RelationshipComponent
{
	entt::entity parent = entt::null;
	std::vector<entt::entity> children;

	RelationshipComponent() = default;
	RelationshipComponent(const RelationshipComponent &) = default;
};
```

Scene Hierarchy에서 Entity를 클릭했을 때 부모 Entity가 자식을 가지고 있다면, 자식 Entity들의 이름도 표시되게 했다.

```cpp
// If node opened, draw child nodes
if (opened)
{
	auto &relation = entity.getComponent<RelationshipComponent>();
	for (auto child : relation.children)
	{
		Entity entity(child, m_Context.get());
		drawEntityNode(entity);
	}
	ImGui::TreePop();
}
```

한 Entity를 다른 Entity의 자식으로 넣거나, Entity가 소유하고 있는 자식을 없애는 등 Relationship을 업데이트하는 기능을 드래그 & 드롭으로 구현했다.

ImGui는 내부적으로 UI들을 서로 구분하기 위해 ID를 사용하는데, 방법이 여러가지가 있다. 텍스트(label)을 ID로 쓰거나, PushID/PopID로 명시적으로 감싸거나 직접 문자열을 ID로 지정할 수 있다.
3가지 방법을 모두 사용해보았는데, 상황마다 좋은 방법이 다르다고 생각했다. 문자열의 경우는 비효율적이지 않나라는 생각이 들었는데, 내부에서 해싱해 관리한다고 하니 괜찮다고 한다.

``` cpp
if (ImGui::BeginDragDropSource())
{
	Entity e = entity;
	// 문자열("EntityPayload")로 ID 설정
	ImGui::SetDragDropPayload("EntityPayload", &e, sizeof(Entity));
	ImGui::Text("%s", tag.c_str());
	ImGui::EndDragDropSource();
}

if (ImGui::BeginDragDropTarget())
{
	// "EntityPayload" 로 드래그 앤 드롭 UI 식별.
	if (const ImGuiPayload *payload = ImGui::AcceptDragDropPayload("EntityPayload"))
	{
		Entity *droppedEntity = (Entity *)payload->Data;
		// droppedEntity가 entity의 자식이 되도록 설정

		if (*droppedEntity != entity)
			updateRelationship(entity, *droppedEntity);
	}
	ImGui::EndDragDropTarget();
}

void SceneHierarchyPanel::updateRelationship(Entity &newParent, Entity &child)
{
	// 1. 기존 부모에게서 제거
	auto &childRelation = child.getComponent<RelationshipComponent>();
	entt::entity oldParent = childRelation.parent;

	if (oldParent != entt::null)
	{
		auto &oldParentRelation = m_Context->m_Registry.get<RelationshipComponent>(oldParent);
		auto &siblings = oldParentRelation.children;
		siblings.erase(std::remove(siblings.begin(), siblings.end(), child), siblings.end());
	}

	// 2. 새 부모로 교체
	childRelation.parent = (entt::entity)newParent;

	// 3. 새 부모의 children 목록에 추가
	auto &parentRelation = newParent.getComponent<RelationshipComponent>();
	parentRelation.children.push_back((entt::entity)child);

	// 4. 자식의 위치를 부모 기준 local좌표로 변환
	auto &tc = child.getComponent<TransformComponent>();
	glm::mat4 parentWorld = newParent.getComponent<TransformComponent>().m_WorldTransform; // 부모의 월드 매트릭스
	glm::mat4 childWorld = child.getComponent<TransformComponent>().m_WorldTransform;	   // 자식의 현재 월드 매트릭스
	glm::mat4 childLocalMat = glm::inverse(parentWorld) * childWorld;

	decomposeMatrix(childLocalMat, tc.m_Scale, tc.m_Rotation, tc.m_Position);
}

```

#### 3. 각종 편의 기능(3D Object, Entity 추가, 삭제)
Entity를 Scene 상에 쉽게 추가하고 삭제할 수 있게 구현했다. 또한 기본 도형(Sphere, Box, Plane, Cylinder, Capsule)도 쉽게 추가할 수 있게 기능을 만들었다. 원래라면 Entity 생성, MeshRendererComponent 추가, meshtype 설정 이렇게 과정을 거쳐야 하는데, 한 번에 원하는 도형을 볼 수 있게 구현했다.

```cpp

// 오른쪽 클릭으로 Popup 활성화
	if (ImGui::BeginPopupContextWindow(0, ImGuiPopupFlags_MouseButtonRight | ImGuiPopupFlags_NoOpenOverItems))
	{
		if (ImGui::MenuItem("Create Empty"))
			m_Context->createEntity("Empty");

		if (ImGui::BeginMenu("3D Object"))
		{
			if (ImGui::MenuItem("Box"))
				m_Context->createPrimitiveMeshEntity("Box", 1);
			if (ImGui::MenuItem("Sphere"))
				m_Context->createPrimitiveMeshEntity("Sphere", 2);
			if (ImGui::MenuItem("Plane"))
				m_Context->createPrimitiveMeshEntity("Plane", 3);
			if (ImGui::MenuItem("Ground"))
				m_Context->createPrimitiveMeshEntity("Ground", 4);
			if (ImGui::MenuItem("Capsule"))
				m_Context->createPrimitiveMeshEntity("Capsule", 5);
			if (ImGui::MenuItem("Cylinder"))
				m_Context->createPrimitiveMeshEntity("Cylinder", 6);
			ImGui::EndMenu();
		}
		ImGui::EndPopup();
	}

Entity Scene::createPrimitiveMeshEntity(const std::string &name, uint32_t idx)
{
	Entity entity = createEntity(name);

	auto &mc = entity.addComponent<MeshRendererComponent>();
	
	// Scene이 생성될 때, 기본 도형을 만들어 보관한다.
	std::shared_ptr<Model> &model = getDefaultModel(idx);

	mc.type = idx;
	mc.m_RenderingComponent = RenderingComponent::createRenderingComponent(model);
	mc.cullSphere = mc.m_RenderingComponent->getCullSphere();

	insertEntityInCullTree(entity);

	return entity;
}

```

#### Inspector
Inspector는 Entity가 갖고 있는 Component들을 자세히 확인할 수 있는 UI이다. Component들을 그리는 UI에 관한 코드를 작성하다보면, 반복적으로 생기는 코드가 많았다. 이를 줄이기 위해 템플릿 함수를 적극적으로 활용했다.

``` cpp

// drawEntityNode에서 작성

ImGui::Begin("Inspector");
if (m_SelectionContext)
{
	drawComponents(m_SelectionContext);
}
ImGui::End();

```

템플릿을 사용했음에도 불구하고, drawComponents() 함수는 약 1000줄이 된다. 이는 반복적으로 사용하는 UI를 따로 함수로 만들지 않거나, Component마다 사용하는 UI를 모듈화하지 않고 한 군데에 모두 작성했기 때문일 것이다. 간단히 맥락만 소개하겠다.

#### Tag & Checkbox

``` cpp

void SceneHierarchyPanel::drawComponents(Entity entity)
{
	if (entity.hasComponent<TagComponent>())
	{
		auto &tc = entity.getComponent<TagComponent>();
		auto &tag = tc.m_Tag;

		char buffer[256];
		memset(buffer, 0, sizeof(buffer));
		strncpy_s(buffer, sizeof(buffer), tag.c_str(), sizeof(buffer));

		std::string label = "##Tag" + std::to_string(entity.getUUID());

		if (ImGui::Checkbox("##Active", &tc.m_selfActive))
		{
			// 자식 엔티티일 경우 부모의 effective 활성 상태를 사용하여 업데이트
			bool parentEffectiveActive = true;
			auto &rc = entity.getComponent<RelationshipComponent>();
			if (rc.parent != entt::null)
			{
				Entity parent{rc.parent, m_Context.get()};
				parentEffectiveActive = parent.getComponent<TagComponent>().m_isActive;
			}
			updateActiveInfo(entity, parentEffectiveActive);
		}

		ImGui::SameLine();

		// ## 뒤의 text는 ImGui 내부의 ID로 사용.
		if (ImGui::InputText(label.c_str(), buffer, sizeof(buffer)))
		{
			tag = std::string(buffer);
		}
	}
	// ...
}

```

Inspector의 가장 위에는 Entity의 이름과 Checkbox가 있다. Checkbox는 해당 Entity가 Scene상에 표시될 것인지 확인하는 Box이다. 기본 값은 True로 설정되어 있다. Hierarchy 상에서 가장 윗단의 부모의 Checkbox가 비활성화 된다면, 자식들도 모두 비활성화 되어야 한다. updateActiveInfo() 함수를 통해 재귀적으로 구현했다.

#### Add Component

``` cpp
void SceneHierarchyPanel::drawComponents(Entity entity)
{
	//...

	if (ImGui::Button("Add Component"))
	{
		ImGui::OpenPopup("AddComponent");
	}

	ImGui::SetNextWindowSizeConstraints(ImVec2(300, 200), ImVec2(600, 400));
	if (ImGui::BeginPopup("AddComponent"))
	{
		const char *text = "Component";
		float windowWidth = ImGui::GetWindowSize().x;
		float textWidth = ImGui::CalcTextSize(text).x;
		float textPosX = (windowWidth - textWidth) * 0.5f;

		ImGui::SetCursorPosX(textPosX);
		ImGui::Text("%s", text);
		ImGui::Separator();

		displayAddComponentEntry<CameraComponent>("Camera");
		displayAddComponentEntry<ScriptComponent>("Script");
		displayAddComponentEntry<MeshRendererComponent>("Mesh Renderer");
		displayAddComponentEntry<LightComponent>("Light");
		displayAddComponentEntry<RigidbodyComponent>("Rigidbody");
		displayAddComponentEntry<SkeletalAnimatorComponent>("Animator");

		bool hasCollider = m_SelectionContext.hasComponent<BoxColliderComponent>() ||
						   m_SelectionContext.hasComponent<SphereColliderComponent>() ||
						   m_SelectionContext.hasComponent<CapsuleColliderComponent>() ||
						   m_SelectionContext.hasComponent<CylinderColliderComponent>();

		if (!hasCollider)
		{
			displayAddComponentEntry<BoxColliderComponent>("Box Collider");
			displayAddComponentEntry<SphereColliderComponent>("Sphere Collider");
			displayAddComponentEntry<CapsuleColliderComponent>("Capsule Collider");
			displayAddComponentEntry<CylinderColliderComponent>("Cylinder Collider");
		}

		ImGui::EndPopup();
	}
	ImGui::PopItemWidth();

	//...
}

template <typename T> void SceneHierarchyPanel::displayAddComponentEntry(const std::string &entryName)
{
	if (!m_SelectionContext.hasComponent<T>())
	{
		if (ImGui::Selectable(entryName.c_str()))
		{
			m_SelectionContext.addComponent<T>();
			ImGui::CloseCurrentPopup();
		}
	}
}

```

Entity에 Component를 쉽게 추가할 수 있게 AddComponent Popup UI를 만들었다. Popup은 Component를 종류별로 추가할 수 있게 표시되고, Collider가 있다면 다른 종류의 Collider를 추가할 수 없게 구현했다. 여기서 displayAddComponentEntry는 반복되는 UI 코드를 줄이기 위해 
템플릿으로 구현했다.

#### draw Component
이제 Component 별로 UI를 작성해야 하는데, 기본 형식은 다음과 같다.

``` cpp

void SceneHierarchyPanel::drawComponents(Entity entity)
{

	drawComponent<ComponentType>("ComponentType", entity, [](auto &component) {
		// 그리고자 하는 UI
	});
}

template <typename T, typename UIFunction>
static void drawComponent(const std::string &name, Entity entity, UIFunction uiFunction)
{
	const ImGuiTreeNodeFlags treeNodeFlags = ImGuiTreeNodeFlags_DefaultOpen | ImGuiTreeNodeFlags_Framed |
											 ImGuiTreeNodeFlags_SpanAvailWidth | ImGuiTreeNodeFlags_AllowItemOverlap |
											 ImGuiTreeNodeFlags_FramePadding;
	if (entity.hasComponent<T>())
	{
		auto &component = entity.getComponent<T>();
		ImVec2 contentRegionAvailable = ImGui::GetContentRegionAvail();

		ImGui::PushStyleVar(ImGuiStyleVar_FramePadding, ImVec2{4, 4});
		float lineHeight = GImGui->Font->FontSize + GImGui->Style.FramePadding.y * 2.0f;
		ImGui::Separator();
		bool open = ImGui::TreeNodeEx((void *)typeid(T).hash_code(), treeNodeFlags, name.c_str());
		ImGui::PopStyleVar();

		// 컴파일 타임에 타입 비교 수행
		if (!std::is_same<T, TransformComponent>::value)
		{
			ImGui::SameLine(contentRegionAvailable.x - lineHeight * 0.5f);
			if (ImGui::Button("+", ImVec2{lineHeight, lineHeight}))
			{
				ImGui::OpenPopup("ComponentSettings");
			}
		}

		bool removeComponent = false;
		if (ImGui::BeginPopup("ComponentSettings"))
		{
			if (ImGui::MenuItem("Remove component"))
				removeComponent = true;

			ImGui::EndPopup();
		}

		if (open)
		{
			uiFunction(component);
			ImGui::TreePop();
		}

		if (removeComponent)
			entity.removeComponent<T>();
	}
}

```

람다함수로 그리고자 하는 UI의 내용(UIFunction)을 작성하면 된다. 아주 간단한 예시 하나만 설명하겠다. 

``` cpp
drawComponent<CylinderColliderComponent>("CylinderCollider", entity, [](auto &component) {

	drawVec3Control("Center", component.m_Center);
	drawFloatControl("Radius", component.m_Radius);
	drawFloatControl("Radius", component.m_Height);
	drawCheckBox("IsTrigger", component.m_IsTrigger);

});

```

CylinderCollider의 경우는 자주 사용하는 UI(vec3, Float, CheckBox)에 관한 것만 있어 내용이 단순하다. 하지만 MeshRendererComponent나 SkeletalAnimatorComponent, ScriptComponent 같은 경우는 작성해야 할 UI의 내용이 매우 많기 때문에 복잡하다. 처음에는 표시해야 할 내용이 많이 없어 하나의 cpp 파일에 작성하는 것이 큰 문제가 되지 않을 것이라 생각했는데, 나중에 다 완성하고 보니 너무 코드가 많고 지저분해 보였다. 다시 Editor를 만들게 될 일이 있다면, 코드를 좀 더 체계적으로 작성해야 할 것 같다.

#### Content Browser
Content Browser는 게임 엔진 안에서 모든 리소스를 관리하는 도구이다. 주요 기능은 폴더 구조 탐색, 미리보기, 검색, 에셋 생성/삭제/이동 등이 있다. 내가 구현한 것은 폴더 구조 탐색, 드래그 & 드롭으로 material이나 texture, skybox 수정하기 기능이다.

