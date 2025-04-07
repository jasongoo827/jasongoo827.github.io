---
title: Game Engine Architecture - Project Settings
tags: [Game Engine, CMake, Makefile]
style: border
color: success
description: ALEngine Project Settings 에 대한 글. CMake로 프로젝트 빌드를 어떻게 했는지 설명.
---

## Project Settings
프로젝트 환경 설정에 대한 글을 따로 쓰게 된 이유는, CMake라는 빌드 도구를 처음 써보며 배우게 된 점이 많기 때문이다. 이전에 42Seoul에서 과제를 할 때는 Makefile을 써 본게 전부이고, 42Seoul 전에는 Visual Studio의 빌드 도구를 썼었다. CMake 말고도 여러모로 배운 점들에 대해 상세히 기록해보려고 한다.

## CMake
CMake는 C++이나 C 같은 컴파일 언어 프로젝트에서 빌드 시스템을 자동화하기 위해 사용되는 도구이다. 쉽게 말해서, 다양한 운영체제와 컴파일러에서 소스 코드를 컴파일하고 실행 파일을 만들 수 있도록 도와주는 설정 도구이다. 

![CMake](https://cmake.org/wp-content/uploads/2023/08/CMake-Logo.svg)

### Why CMake?
CMake를 쓰는 이유를 간단하게 정리해보면 다음과 같다.

#### 1. **플랫폼/컴파일러 독립적인 빌드 시스템 생성**
- 하나의 CMake 설정 파일(`CMakeLists.txt`)만 있으면,
  - Windows에선 Visual Studio 프로젝트 생성
  - Linux에선 Makefile 생성
  - macOS에선 Xcode 프로젝트 생성
- 즉, 여러 운영체제에서 코드 하나로 크로스 플랫폼 개발이 가능해진다.

#### 2. **빌드 설정 자동화**
- 수동으로 많은 컴파일 명령어를 처리하는 것은 비효율적이다.
- CMake는 소스 파일, 라이브러리, 컴파일 옵션 등을 자동으로 처리해준다.
- 복잡한 디렉터리 구조, 의존성, 모듈도 쉽게 다룰 수 있다.

#### 3. **프로젝트 규모가 커질수록 효과적**
- **수십, 수백 개 파일이 있는 게임 프로젝트**에서는 CMake가 거의 필수이다.
- 모듈별로 타겟을 나누고, 의존성을 명확히 하여 유지보수와 협업이 쉬워짐.

#### 4. **외부 라이브러리 연동 간편**
- `find_package`, `target_link_libraries` 등을 통해 쉽게 연결 가능.
- `FetchContent` 기능을 쓰면 GitHub에서 직접 다운로드해서 빌드까지 가능.

#### 5. **Out-of-source build**
- 소스 디렉터리와 빌드 디렉터리를 분리 가능해서, 깔끔한 프로젝트 관리가 가능.
- 예: `build/` 폴더 하나만 지우면 깨끗한 상태로 되돌릴 수 있음.

---

## 프로젝트 파일 구조
프로젝트는 AL, AL-ScriptCore, Sandbox, SandboxProject 총 4가지로 구성되어 있다. 각 프로젝트에 대해 설명하며 CMake 파일을 어떻게 작성했고, 그 과정에서 무엇을 고려했고 어떤 점이 어려웠는지 작성해보겠다.

### AL
AfterLife Engine의 코어 시스템. Window, EventSystem, Profiler, Renderer, Physics, Memory, Scene, Scripting 등 코어한 모든 기능이 포함되어 있는 프로젝트이다. 

```text
AL/include
├── Core
├── Debug
├── Events
├── ImGui
├── Memory
├── Physics
├── Platform
├── Project
├── Renderer
├── Scene
├── Scripting
├── UI
└── Utils
```

### CMakeLists.txt
어차피 CMake 파일 명령어 사용법은 반복되는 부분이 많으니 이 부분에서 설명하고 넘어가겠다. AL의 CMake 파일을 보면서 설명하겠다.


```
cmake_minimum_required(VERSION 3.20)
set(PROJECT_NAME AfterLife)
project(${PROJECT_NAME})

# Set Macros
set(WINDOW_NAME "ALEngine")
set(WINDOW_WIDTH 1920)
set(WINDOW_HEIGHT 1080)

# 의존성 설정
include(${CMAKE_SOURCE_DIR}/AL/Dependency.cmake)

# 소스 파일 및 헤더 파일 검색
file(GLOB_RECURSE SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp")
file(GLOB_RECURSE HEADERS "${CMAKE_CURRENT_SOURCE_DIR}/include/*.h")

# AL 라이브러리 생성
add_library(${PROJECT_NAME} STATIC ${SOURCES}) # SOURCES만 전달

# PCH 설정
target_precompile_headers(${PROJECT_NAME} PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/include/alpch.h)

# VulkanSDK 설정 - 설치 여부 확인 필요. 버전 확인 필요.
set(CMAKE_PREFIX_PATH "C:/VulkanSDK/1.3.296.0")
find_package(Vulkan REQUIRED)
target_link_libraries(${PROJECT_NAME} PUBLIC Vulkan::Vulkan)

# 헤더 파일 경로 포함
target_include_directories(${PROJECT_NAME} PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
target_include_directories(${PROJECT_NAME} PUBLIC ${DEP_INCLUDE_DIR})

# 시스템 라이브러리 추가 - 최소 환경 조건으로 명시하기(Visual Studio - Window SDK 설치)
set(SYSTEM_LIBS WS2_32.lib WinMM.lib Version.lib Bcrypt.lib)
target_link_libraries(${PROJECT_NAME} PRIVATE ${SYSTEM_LIBS})

# lib 경로 설정
target_link_directories(${PROJECT_NAME} PUBLIC ${DEP_LIB_DIR})
target_link_libraries(${PROJECT_NAME} PUBLIC ${DEP_LIBS})

# AL의 컴파일 옵션 설정 (필요에 따라 추가)
target_compile_options(${PROJECT_NAME} PUBLIC "/utf-8")

# 매크로 정의
target_compile_definitions(${PROJECT_NAME} PUBLIC
WINDOW_NAME="${WINDOW_NAME}"
WINDOW_WIDTH=${WINDOW_WIDTH}
WINDOW_HEIGHT=${WINDOW_HEIGHT})

# 출력 디렉토리 설정
set_target_properties(${PROJECT_NAME} PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
    LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
    ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
)

# Dependency들이 먼저 build 될 수 있게 관계 설정 / 뒤에서 부터 컴파일
add_dependencies(${PROJECT_NAME} ${DEP_LIST})

```

### 사용된 명령어

- cmake_minimum_required : 사용하려는 최소 CMake 버전을 지정
- set: 변수 설정
- project: 프로젝트 이름 설정

- include: 의존성 설정. 보통 헤더 파일이나 Dependency 파일을 지정한다.
- CMAKE_SOURCE_DIR: 소스파일의 가장 top level의 경로
- GLOB_RECURSE: 하위 디렉터리를 포함하여 모든 파일을 재귀적으로 탐색
- CMAKE_CURRENT_SOURCE_DIR: 현재 소스 경로

- add_library: 라이브러리 생성
- STATIC: 정적 라이브러리 옵션. SHARED를 사용하면 동적 라이브러리가 생긴다.

- target_precompile_headers: pch 설정

- find_package: 외부 라이브러리나 모듈을 찾음. 경로나 매개변수를 설정해 줘야 함
- target_link_libraries: 프로젝트에 라이브러리 연결

- target_include_directories: 헤더 파일 경로 지정
- target_compile_options: 컴파일 옵션 설정
- target_compile_definitions: 매크로 변수 설정

- set_target_properties: 빌드 결과물의 경로를 명시적으로 지정해 정리된 구조를 유지

- add_dependencies: 외부 라이브러리들이 먼저 빌드되도록 명시적 빌드 순서를 지정

대부분 Makefile을 만들어보며 아는 설정들이었지만 새롭게 알게 된 것은 GLOB_RECURSE, target_precompile_headers 이다. Mac에서 Makefile을 작성할 때 보통 Wildcard(*)를 사용해 *.cpp 또는 *.h를 찾아왔었는데, CMake에도 이 기능을 하는 명령어가 있어서 좋았다. 하지만 실시간으로 파일이 추가/삭제 되는 경우는 잘 처리 못하니 주의해야 한다. PCH에 대해서는 처음 알게 됐다. PCH는 컴파일 속도를 향상하기 위해 자주 변하지 않는 헤더 파일들을 미리 컴파일해두고, 나중에 재사용하는 방식이다. STL의 무거운 헤더들이나 Windows.h 같은 헤더들을 미리 컴파일 해두면 빌드 속도를 향상할 수 있다.

### Dependency.cmake
CMake를 처음 써 본 입장에서 이게 가장 좋았다. 외부 라이브러리들을 미리 프로젝트에 포함시키는 방식이 아니라, 실시간으로 외부에서 가져온 후 build 디렉토리 아래에 include 설정과 링크만 해주면 사용할 수 있는 점이 너무 편했다. 프로젝트의 용량을 줄일 수 있는 아주 좋은 도구이다. 대부분의 라이브러리들은 git 주소를 통해 가져왔다.

```
# ExternalProject 관련 명령어 셋 추가
include(ExternalProject)

# Dependency 관련 변수 설정
set(DEP_INSTALL_DIR ${PROJECT_BINARY_DIR}/install)
set(DEP_INCLUDE_DIR ${DEP_INSTALL_DIR}/include)
set(DEP_LIB_DIR ${DEP_INSTALL_DIR}/lib)

# glfw
ExternalProject_Add(
    dep_glfw
    GIT_REPOSITORY "https://github.com/glfw/glfw.git"
    GIT_TAG "3.3.2"
    GIT_SHALLOW 1
    UPDATE_COMMAND "" PATCH_COMMAND "" TEST_COMMAND ""
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX=${DEP_INSTALL_DIR}
        -DGLFW_BUILD_EXAMPLES=OFF
        -DGLFW_BUILD_TESTS=OFF
        -DGLFW_BUILD_DOCS=OFF
)
set(DEP_LIST ${DEP_LIST} dep_glfw)
set(DEP_LIBS ${DEP_LIBS} glfw3)

# spdlog
ExternalProject_Add(
    dep_spdlog
    GIT_REPOSITORY "https://github.com/gabime/spdlog.git"
    GIT_TAG "v1.11.0"
    GIT_SHALLOW 1
    UPDATE_COMMAND "" PATCH_COMMAND "" TEST_COMMAND ""
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX=${DEP_INSTALL_DIR}
        -DSPDLOG_BUILD_EXAMPLES=OFF
        -DSPDLOG_BUILD_TESTS=OFF
        -DSPDLOG_BUILD_BENCH=OFF
        -DSPDLOG_BUILD_SHARED=OFF
)

set(DEP_LIST ${DEP_LIST} dep_spdlog)
set(DEP_LIBS ${DEP_LIBS} 
    $<$<CONFIG:Debug>:spdlogd>
    $<$<CONFIG:Release>:spdlog>
)

# imgui 추가
ExternalProject_Add(
    dep_imgui
    GIT_REPOSITORY "https://github.com/Very-Real-Engine/imgui.git"
    GIT_TAG "main"
    GIT_SHALLOW 1
    UPDATE_COMMAND "" PATCH_COMMAND "" TEST_COMMAND ""
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX=${DEP_INSTALL_DIR}
)
set(DEP_LIST ${DEP_LIST} dep_imgui)
set(DEP_LIBS ${DEP_LIBS} imgui)

# glm
ExternalProject_Add(
	dep_glm
	GIT_REPOSITORY "https://github.com/g-truc/glm"
	GIT_TAG "0.9.9.8"
	GIT_SHALLOW 1
	UPDATE_COMMAND ""
	PATCH_COMMAND ""
    CMAKE_ARGS
        -DGLM_TEST_ENABLE=OFF  # 🔥 GLM 테스트 코드 비활성화
	TEST_COMMAND ""
	INSTALL_COMMAND ${CMAKE_COMMAND} -E copy_directory
		${PROJECT_BINARY_DIR}/dep_glm-prefix/src/dep_glm/glm
		${DEP_INSTALL_DIR}/include/glm
	)
set(DEP_LIST ${DEP_LIST} dep_glm)

# stb
ExternalProject_Add(
	dep_stb
	GIT_REPOSITORY "https://github.com/nothings/stb"
	GIT_TAG "master"
	GIT_SHALLOW 1
	UPDATE_COMMAND ""
	PATCH_COMMAND ""
	CONFIGURE_COMMAND ""
	BUILD_COMMAND ""
	TEST_COMMAND ""
	INSTALL_COMMAND ${CMAKE_COMMAND} -E copy
		${PROJECT_BINARY_DIR}/dep_stb-prefix/src/dep_stb/stb_image.h
		${DEP_INSTALL_DIR}/include/stb/stb_image.h
	)
set(DEP_LIST ${DEP_LIST} dep_stb)

# assimp
ExternalProject_Add(
	dep_assimp
	GIT_REPOSITORY "https://github.com/assimp/assimp"
	GIT_TAG "v5.4.3"
	GIT_SHALLOW 1
	UPDATE_COMMAND ""
	PATCH_COMMAND ""
	CMAKE_ARGS
		-DCMAKE_INSTALL_PREFIX=${DEP_INSTALL_DIR}
        -DCMAKE_BUILD_TYPE=$<CONFIG>
		-DBUILD_SHARED_LIBS=OFF
		-DASSIMP_BUILD_ASSIMP_TOOLS=OFF
		-DASSIMP_BUILD_TESTS=OFF
		-DASSIMP_INJECT_DEBUG_POSTFIX=ON
		-DASSIMP_BUILD_ZLIB=ON
	TEST_COMMAND ""
)
set(DEP_LIST ${DEP_LIST} dep_assimp)
set(DEP_LIBS ${DEP_LIBS}
    $<$<CONFIG:Debug>:assimp-vc143-mtd>
    $<$<CONFIG:Release>:assimp-vc143-mt>
    $<$<CONFIG:Debug>:zlibstaticd>
    $<$<CONFIG:Release>:zlibstatic>
)
	
# yaml-cpp
ExternalProject_Add(
    dep_yaml_cpp
    GIT_REPOSITORY "https://github.com/jbeder/yaml-cpp.git"
    GIT_TAG "yaml-cpp-0.7.0" 
    GIT_SHALLOW 1
    UPDATE_COMMAND ""
    PATCH_COMMAND ""
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX=${DEP_INSTALL_DIR}
        -DYAML_BUILD_SHARED_LIBS=OFF
        -DYAML_CPP_BUILD_TESTS=OFF
        -DYAML_CPP_BUILD_TOOLS=OFF
    TEST_COMMAND ""
)
set(DEP_LIST ${DEP_LIST} dep_yaml_cpp)
set(DEP_LIBS ${DEP_LIBS}
    $<$<CONFIG:Debug>:yaml-cppd>
    $<$<CONFIG:Release>:yaml-cpp>
)

# Mono
ExternalProject_Add(
    dep_mono
    GIT_REPOSITORY "https://github.com/Very-Real-Engine/ALE_mono.git"
    GIT_TAG "main"
    GIT_SHALLOW 1
    UPDATE_COMMAND ""
    PATCH_COMMAND ""
    CONFIGURE_COMMAND "" 
    BUILD_COMMAND ""     
    INSTALL_COMMAND      
        ${CMAKE_COMMAND} -E copy_directory
        <SOURCE_DIR>/mono  
        ${DEP_INCLUDE_DIR}/mono
        COMMAND ${CMAKE_COMMAND} -E copy
        <SOURCE_DIR>/Debug/libmono-static-sgen.lib
        ${DEP_INSTALL_DIR}/lib/libmono-static-sgen-debug.lib
        COMMAND ${CMAKE_COMMAND} -E copy
        <SOURCE_DIR>/Release/libmono-static-sgen.lib
        ${DEP_INSTALL_DIR}/lib/libmono-static-sgen.lib
)

# Dependency 리스트 추가
set(DEP_LIST ${DEP_LIST} dep_mono)
# Mono 라이브러리 경로 설정
set(MONO_LIB_DEBUG ${DEP_INSTALL_DIR}/lib/libmono-static-sgen-debug.lib)
set(MONO_LIB_RELEASE ${DEP_INSTALL_DIR}/lib/libmono-static-sgen.lib)

# CMake에서 빌드 타입에 따라 올바른 라이브러리를 링크
set(DEP_LIBS ${DEP_LIBS} 
    $<$<CONFIG:Debug>:${MONO_LIB_DEBUG}>
    $<$<CONFIG:Release>:${MONO_LIB_RELEASE}>
)
```

내가 해줘야 했던 일은 각 라이브러리에 포함된 CMake 파일을 파악해, 어떤 옵션으로 빌드해야 할지 정해주는 것이었다. 또한 Debug, Release 빌드에 따라 다르게 설정해줘야 했다. 대부분 반복되는 작업이어서 크게 어렵지는 않았다. 까다로웠던 점은 두 가지 정도 있는데, 먼저 imgui나 mono는 CMake 파일이 없어서 직접 작성해줘야 했던 점이다. 당연히 CMake가 있는 줄 알고 빌드를 하다가 계속 오류가 생겨 헤맸던 기억이 있다. 또 까다로웠던 점은 glm은 CMAKE_ARGS -DGLM_TEST_ENABLE=OFF 이 옵션이 다른 라이브러리의 테스트 코드 비활성화 약간 달랐던 점이다. 그냥 무지성으로 옵션들을 복사 붙여넣기 하다가 혼났던 기억이 있다.

### AL-ScriptCore
ScriptCore는 Scripting의 규칙을 명시해 놓은 프로젝트이다.

```
AL-ScriptCore/
└── src
    ├── Input.cs
    ├── InternalCalls.cs
    ├── KeyCode.cs
    ├── Scene
    ├── Vector2.cs
    ├── Vector3.cs
    └── Vector4.cs
```

이 프로젝트는 .NET 기반 C# 프로젝트이기 때문에 약간 다르게 설정해줘야 했다. Debug/Release 별로 Visual Studio에서 어떻게 build 할지 옵션을 설정해줬다.

### CMakeLists.txt
```
# Debug / Release 빌드에 따라 값 설정
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    # Visual StudioDebug 빌드 설정
    set_target_properties(AL-ScriptCore PROPERTIES
        VS_GLOBAL_Optimize "false"
        VS_GLOBAL_DebugSymbols "true"
        VS_DOTNET_TARGET_FRAMEWORK_VERSION ${DOTNET_FRAMEWORK_VERSION}
    )
else()
    set_target_properties(AL-ScriptCore PROPERTIES
        VS_GLOBAL_Optimize "true"
        VS_GLOBAL_DebugSymbols "false"
        VS_DOTNET_TARGET_FRAMEWORK_VERSION ${DOTNET_FRAMEWORK_VERSION}
    )
endif()
```

### Sandbox
Sandbox는 Editor의 소스코드 및 엔진에 필요한 각종 에셋(폰트, Skybox)과 자원(Play, Stop, Pause 버튼 이미지, File Icon), SandboxProjects 들이 모여있는 프로젝트이다. 

```
Sandbox/
├── Project
├── Resources
├── assets
├── mono
└── src
```

### Sandbox Project
Sandbox Project는 엔진으로 생성된 프로젝트이다. Unity를 떠올려봤을 때 Unity Hub에서 원하는 프로젝트를 설정해 생성할 수 있는 것처럼, ALEngine도 Sandbox 아래에 정해놓은 형식대로 폴더와 Scene 파일을 배치하면 프로젝트를 생성할 수 있다. 환경 설정을 하며 가장 아쉬웠던 부분인데, 가장 위에 배치되어 있는 CMake 파일에서 설정만 잘 해놓거나 직접 프로젝트를 만들 수 있는 Unity Hub 같은 도구를 만들었다면 프로젝트 생성을 자동화 할 수 있었을 텐데... 그 부분이 참 아쉽다. 이 프로젝트의 CMake는 게임 코드를 빌드하는 역할을 수행한다.

```
Sandbox/Project
└── Assets
    ├── Models
    ├── Scenes
    └── Scripts
```

### Main CMakeLists.txt
가장 처음에 실행되는 CMake이다. 윈도우가 아닐 때는 실행이 안되게 처리해 놓았다. 만약에 Cross Platform을 지원한다면 이 부분을 좀 더 자세하게 작성해야 할 것이다.

```
cmake_minimum_required(VERSION 3.20)
project(GameEngine)

if(NOT WIN32)
    message(FATAL_ERROR "This project can only be built on Windows.")
endif()

message(STATUS "Building on Windows environment.")

# C++ 표준 설정
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(SANDBOXPROJECT "Sandbox/Project/Assets/Scripts")

# 서브 디렉토리 추가
add_subdirectory(AL-ScriptCore)
add_subdirectory(AL)
add_subdirectory(${SANDBOXPROJECT})
add_subdirectory(Sandbox)
```

## 느낀 점
Makefile이 아닌 CMake를 사용해 빌드해본 것은 처음이었는데, 참 편한 점이 많다고 느꼈다. 프로젝트 진행하는 동안 팀원들 대부분 문제 없이 빌드를 할 수 있었다. 나는 기본적인 기능만 하는 CMake를 작성했지만, 모든 환경에서 돌아가는 프로그램을 작성하려면 CMake도 환경별로 분기를 엄청 자세하게 나누고, 예외 처리도 잘 해야겠다는 생각이 들었다. 나중에 또 CMake를 작성할 일이 있다면 병렬 설정을 통해 빌드 속도를 높이는 작업도 해보고 싶다.