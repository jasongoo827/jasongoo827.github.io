---
title: Scripting
tags: [Game Engine, Scripting]
style: border
color: success
description: ALEngine의 Scene system, gui 에 대한 글.
---

## Scripting
지난 포스트는 ALEngine의 Scene Load System, Editor에 대한 설명을 했다. 오늘은 Scripting System을 어떻게 구현했는지에 대한 설명을 하려고 한다. 게임 엔진에서 Scripting은 게임의 로직이나 이벤트, 상호작용 등을 엔진의 핵심 코드와 분리하여 작성할 수 있게 해주는 시스템을 의미한다. 일반적으로 Scripting은 C++처럼 성능 중심의 언어로 작성된 엔진 내부 로직과 달리, Python, Lua, C#, JavaScript 같은 비교적 간단하고 동적인 언어를 사용하여 구현된다. 대표적인 예로 Unity는 C#을, Unreal Engine은 Blueprint나 C++ 기반의 Scripting을 지원하여 사용자 정의 로직을 쉽게 작성할 수 있게 돕고 있다. ALEngine은 Mono를 이용해 C# Scripting System을 구현했다.

## Mono
Mono는 .NET Framework의 오픈소스 구현체로, 다양한 플랫폼에서 C#을 실행할 수 있도록 해주는 라이브러리이다. 간단히 말하면 C# 코드를 C/C++로 변환해준다. Mono를 사용하면 다음과 같은 기능을 엔진에 도입할 수 있다.

	• C#으로 작성한 코드 컴파일 및 실행
	• C++ 엔진 코드와 C# 간의 양방향 호출 (바인딩)
	• C# 클래스의 메서드나 속성 접근
	• 스크립트 클래스에서 엔진 Entity에 접근하거나 이벤트 수신

즉, Mono를 통해 컴파일된 외부 C# 스크립트를 엔진의 런타임 동안 로딩하고, 해당 클래스 정보를 활용해 엔진의 Entity와 연결할 수 있도록 설계했다. 이 덕분에 게임 로직은 별도의 스크립트로 작성되며, 엔진은 그것을 실행하는 역할에 집중할 수 있게 되었다.

---

## System Flow

ALEngine의 Scripting System은 다음과 같은 흐름으로 동작한다

**1. C# 스크립트 작성 및 DLL 빌드**

    사용자는 Player.cs, Camera.cs 같은 C# 클래스를 작성한 후, 이를 DLL로 컴파일한다. 이 DLL은 엔진이 실행될 때 로딩된다. 모든 스크립트 클래스는 Entity 클래스를 상속받으며, onCreate(), onUpdate(float ts) 등의 메서드를 구현해야 한다는 규칙을 따른다. 또한 스크립트에서 사용하는 규칙(자료형, 클래스)을 따로 정의해 이를 DLL로 컴파일한다.

    ``` csharp

    namespace Sandbox
    {
        public class Player : Entity
        {
            void onCreate()
            {
            }

            void onUpdate(float ts)
            {
            }
        }
    }

    namespace ALEngine
    {
        public struct Vector2
        {
            public float X, Y;

            public static Vector2 Zero => new Vector2(0.0f);
            
            // ...
        }

        public struct Vector3
        {
            public float X, Y, Z;

            public static Vector3 Zero => new Vector3(0.0f);

            // ...
        }
    }

    ```

**2. 엔진 실행 시 DLL 로딩 및 클래스 등록**

    엔진은 시작 시 C# DLL을 Mono를 통해 로딩하고, 내부에 포함된 모든 클래스 정보를 수집한다. 이 과정에서 클래스 이름, 메서드 포인터, 필드 정보 등이 ScriptClass 객체에 저장된다. 이는 나중에 Entity와 연결될 때 사용된다.

    ``` csharp
    
   void ScriptingEngine::loadAssemblyClasses()
    {
        s_Data->entityClasses.clear();

        // Metadata를 나타내는 구조체 - 타입, 메서드, 필드 정보 등을 포함.
        const MonoTableInfo *typeDefsTable = mono_image_get_table_info(s_Data->appAssemblyImage, MONO_TABLE_TYPEDEF);
        int32_t numTypes = mono_table_info_get_rows(typeDefsTable);
        MonoClass *entityClass = mono_class_from_name(s_Data->coreAssemblyImage, "ALEngine", "Entity");

        for (int32_t i = 0; i < numTypes; ++i)
        {
            uint32_t cols[MONO_TYPEDEF_SIZE]; // 6
            mono_metadata_decode_row(typeDefsTable, i, cols, MONO_TYPEDEF_SIZE);

            const char *nameSpace = mono_metadata_string_heap(s_Data->appAssemblyImage, cols[MONO_TYPEDEF_NAMESPACE]);
            const char *className = mono_metadata_string_heap(s_Data->appAssemblyImage, cols[MONO_TYPEDEF_NAME]);
            std::string fullName;

            // C# Style
            if (strlen(nameSpace) != 0)
                fullName = fmt::format("{}.{}", nameSpace, className);
            else
                fullName = className;

            MonoClass *monoClass = mono_class_from_name(s_Data->appAssemblyImage, nameSpace, className);

            // ScriptCore에 정의한 Class와 일치.
            if (monoClass == entityClass)
                continue;

            // Class 계층 구조 검사, 부모-자식 관계 확인
            bool isEntity = mono_class_is_subclass_of(monoClass, entityClass, false);
            if (!isEntity)
                continue;

            std::shared_ptr<ScriptClass> scriptClass = std::make_shared<ScriptClass>(nameSpace, className);
            s_Data->entityClasses[fullName] = scriptClass;

            // check class fields
            int32_t fieldCount = mono_class_num_fields(monoClass);
            void *iterator = nullptr;
            while (MonoClassField *field = mono_class_get_fields(monoClass, &iterator))
            {
                const char *fieldName = mono_field_get_name(field); // variable name
                uint32_t flags = mono_field_get_flags(field);
                if (flags & MONO_FIELD_ATTR_PUBLIC)
                {
                    MonoType *type = mono_field_get_type(field); // variable type
                    EScriptFieldType fieldType = utils::monoTypeToScriptFieldType(type);
                    scriptClass->m_Fields[fieldName] = {fieldType, fieldName, field};
                }
            }
        }
    }

    ```

**3. ScriptComponent를 가진 Entity 탐색 및 인스턴스 생성**

    Scene이 로드되면, 모든 Entity를 순회하며 ScriptComponent가 있는지를 확인한다. 만약 해당 컴포넌트의 클래스 이름이 사전에 등록된 ScriptClass와 일치한다면, 해당 C# 클래스의 인스턴스를 생성한다.

    ``` cpp

    void ScriptingEngine::onCreateEntity(Entity entity)
    {
        const auto &sc = entity.getComponent<ScriptComponent>();
        if (ScriptingEngine::entityClassExists(sc.m_ClassName))
        {
            UUID entityID = entity.getUUID();

            std::shared_ptr<ScriptInstance> instance =
                std::make_shared<ScriptInstance>(s_Data->entityClasses[sc.m_ClassName], entity);

            s_Data->entityInstances[entityID] = instance;

            // Copy field values
            if (s_Data->entityScriptFields.find(entityID) != s_Data->entityScriptFields.end())
            {
                const ScriptFieldMap &fieldMap = s_Data->entityScriptFields.at(entityID);
                for (const auto &[name, fieldInstance] : fieldMap)
                    instance->setFieldValueInternal(name, fieldInstance.m_Buffer);
            }
            instance->invokeOnCreate();
        }
    }

    ```

**4. 스크립트 메서드 실행**

    생성된 인스턴스에 대해 onCreate()가 한 번 호출되며, 매 프레임마다 onUpdate(ts) 메서드가 호출된다. 이 때 ts는 프레임 간 시간 간격으로, 엔진이 매 프레임 전달한다. ScriptInstance는 특정 Entity에 연결된 C# 인스턴스를 관리한다. 현재는 C++에서 스크립트를 제어하는 단방향 구조이지만, 향후 C# 측에서도 엔진 Entity를 조작할 수 있도록 래퍼 클래스를 제공하는 방식으로 확장할 수 있다.

    ``` cpp

    void ScriptingEngine::onUpdateEntity(Entity entity, Timestep ts)
    {
        UUID entityID = entity.getUUID();

        // Runtime중 삭제될 수 있기 때문에 check needed.
        if (s_Data->entityInstances.find(entityID) != s_Data->entityInstances.end())
        {
            std::shared_ptr<ScriptInstance> instance = s_Data->entityInstances[entityID];
            instance->invokeOnUpdate((float)ts);
        }
        else
        {
            AL_CORE_ERROR("Could not find ScriptInstance for entity {}", entityID);
        }
    }

    ```

**5. C# → C++ 호출: Internal Call을 통한 엔진 API 접근**

    C# 스크립트에서 엔진 기능을 직접 호출할 수 있도록 Mono의 internal call 기능을 사용해 주요 엔진 API를 바인딩했다. 예를 들어 getComponent<T>(), addForce(Vector3 force)와 같은 함수들은 C++ 측 엔진 함수와 연결되어 있으며, 스크립트에서 직접 사용할 수 있다.

    ``` csharp

    namespace ALEngine
    {
        // ...

        public class RigidbodyComponent : Component
        {
            public void addForce(Vector3 force)
            {
                InternalCalls.RigidbodyComponent_addForce(Entity.ID, ref force);
            }
        }

        // ...
    }

    namespace ALEngine
    {
        public static class InternalCalls
        {
            // ...

            #region RigidbodyComponent
            [MethodImplAttribute(MethodImplOptions.InternalCall)]
            internal extern static void RigidbodyComponent_addForce(ulong entityID, ref Vector3 force);
            #endregion
            
            // ...
        }
    }

    ```


    ``` cpp

    // ALEngine API
    static void RigidbodyComponent_addForce(UUID entityID, glm::vec3 *force)
    {
        Scene *scene = ScriptingEngine::getSceneContext();
        Entity entity = scene->getEntityByUUID(entityID);

        Rigidbody *body = (Rigidbody *)entity.getComponent<RigidbodyComponent>().body;
        body->registerForce(*force);
    }


    ```

이 구조를 통해 스크립트 로직과 엔진 내부 로직은 명확히 분리되며, 유연한 게임 로직 작성이 가능해진다.

---

## 마무리 및 회고
ALEngine의 Scripting System은 사전에 작성된 C# 스크립트를 DLL로 컴파일한 후, Mono를 통해 로딩하고 ScriptClass 및 ScriptInstance 구조로 관리하는 방식으로 설계되었다. 이 방식은 런타임 중 안전하고 빠르게 스크립트를 실행할 수 있으며, 엔진 내부와 스크립트 간의 명확한 역할 분리를 가능하게 한다.

또한 Mono의 Internal Call 기능을 활용하여 C#에서 엔진의 핵심 기능을 일부 호출할 수 있도록 함으로써, 스크립트 측에서도 게임 플레이 로직을 유연하게 제어할 수 있다.

다만 몇 가지 부분에서 개선해야 할 부분이 있다.

- 스크립트에서 getComponent<T>()함수 호출 시, 동적 할당을 통한 Component 반환.
- 런타임에 새로운 Script를 작성하거나 기존의 Script 편집 불가능.
- 더 많은 엔진 기능(API)을 제공하지 못함. 특히 렌더링 부분.

이외에도 개선할 점이 많지만 이 정도에서 마무리하려고 한다. 이렇게 길고 길었던 ALEngine에 대한 포스트가 끝났다. 엔진 프로젝트를 진행하면서 많은 것을 배웠고, 나에 대한 한계도 많이 느낄 수 있었다. 이 프로젝트를 좋게 봐주셔서 요새 면접 볼 일이 몇 번 있었는데, 가서 개발자 분들과 얘기를 나누다 보면 아직 나는 멀었구나라는 생각도 하게 된다.

다음 포스트는 42Seoul에서 공부하며 구현한 웹서버에 대해 다룰 것 같다. 요새 게임회사들은 주로 멀티 게임을 만들기 때문에, 클라이언트 개발자도 네트워크에 대한 지식을 필수적으로 갖고 있어야 한다. 웹서버에 대한 포스트 이후에는 면접이나 코딩 테스트를 보며 잘 몰랐던, 내가 간과했던 지식들에 대한 짧은 포스트를 쓸 것이다.

곧 다시 돌아오겠다!
