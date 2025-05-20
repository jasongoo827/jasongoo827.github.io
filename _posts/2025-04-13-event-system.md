---
title: Game Engine Architecture - Input Event System
tags: [Game Engine]
style: border
color: success
description: ALEngine의 Input Event System에 관한 글.
---

## Input Event System

지난 포스트는 ALEngine의 한 프레임에서 Update Loop에 대해 설명했었다. 이번에는 Update Loop의 첫 단계인 입력 처리에 대한 부분이다. 입력 처리 시스템이 따로 필요한 이유는 무엇일까? 

#### 프로그램이 이해할 수 있는 명령으로 처리 가능
게임 장치는 키보드, 마우스, 컨트롤러, 터치스크린 등 다양한 입력 장치로부터 신호를 받는다. 이를 엔진이 이해할 수 있는 이벤트, 명령으로 바꿔준다.

#### 다양한 플랫폼 지원 가능
하나의 게임을 여러 플랫폼(PC, 콘솔, 모바일 등)에서 실행하려면, 입력 방식이 전부 다르다. 입력 처리 시스템이 **추상화**되어 있다면, 각 플랫폼별 입력 방식을 일관된 인터페이스로 통합할 수 있다.

#### 입력 큐 및 이벤트 관리 가능
입력을 실시간으로 처리하는 게 아니라, 일단 큐에 넣고 정해진 순서대로 처리할 수 있다. 이 방식은 프레임 단위로 정확한 처리가 가능해서, 멀티플레이나 고속 액션에서도 입력 지연을 줄여준다.
이러한 여러 가지 이유로 엔진에서 입력 처리 시스템을 구현해놓으면 코드의 유지보수성, 확장성이 좋아지고 엔진을 사용하는 프로그래머나 게임을 하는 플레이어에게 편리함을 제공할 수 있다.

## 코드 설계
나는 MacOS, Linux 등에도 동작하는 확장성 있는 엔진을 만들고 싶었기 때문에, 입력 시스템을 추상화해야 했다. 입력 시스템을 구현하려면 먼저 창(Window)이 필요했다. Vulkan과 Window OS를 모두 지원하는 라이브러리를 조사해보니, GLFW가 있었다.  

#### GLFW
**GLFW**는 Graphics Library Framework의 약자로 OpenGL, OpenGL ES, Vulkan 기반 어플리케이션 개발을 위한 **오픈 소스 크로스 플랫폼 라이브러리**이다. 이 라이브러리는 창 생성, 렌더링 컨텍스트와 서피스 설정, 입력 처리, 이벤트 핸들링 등을 위한 간단하고 플랫폼 독립적인 API를 제공한다. ALEngine은 Vulkan을 기반으로 한 Window 어플리케이션이기 때문에 GLFW를 사용하는 것이 적절했다. 


#### Window Class
ALEngine은 Vulkan, Window만 지원하면 됐지만, 나는 확장성 있는 코드를 작성하고 싶었다. 그래서 Window 클래스를 만들어 추상화 한 후, OS에 맞게 구체화하는 코드를 작성했다.

```cpp
class Window
{
  public:

	using EventCallbackFn = std::function<void(Event &)>;
	virtual ~Window() = default;
	virtual void onUpdate() = 0;
	virtual uint32_t getWidth() const = 0;
	virtual uint32_t getHeight() const = 0;
	virtual GLFWwindow *getNativeWindow() const = 0;
	virtual void setEventCallback(const EventCallbackFn &callback) = 0;
	virtual void setVSync(bool enabled) = 0;
	virtual bool isVSync() const = 0;
	static Window *create(const WindowProps &props = WindowProps());
};
```

#### WindowsWindow Class
Window OS에서 동작하는 Window Class

```cpp
class WindowsWindow : public Window
{
  public:

	WindowsWindow(const WindowProps &props);
	virtual ~WindowsWindow();
	void onUpdate() override;

	inline uint32_t getWidth() const override
	{
		return m_Data.width;
	}
	inline uint32_t getHeight() const override
	{
		return m_Data.height;
	}
	inline void setEventCallback(const EventCallbackFn &callback) override
	{
		m_Data.eventCallback = callback;
	}
	inline GLFWwindow *getNativeWindow() const override
	{
		return m_Window;
	}

	void setVSync(bool enabled) override;
	bool isVSync() const override;

  private:
	virtual void init(const WindowProps &props);
	virtual void shutDown();

  private:
	GLFWwindow *m_Window;

	struct WindowData
	{
		std::string title;
		uint32_t width, height;
		bool vSync;

		EventCallbackFn eventCallback;
	};

	WindowData m_Data;
};
```

WindowsWindow 클래스를 통해 창을 생성하고 나면, 입력 처리를 구현할 수 있다. GLFW에는 특정 이벤트가 발생했을 때, 호출되는 Callback 함수들이 있다. 하나의 예시에 대해 먼저 설명하겠다.

```
// 창의 크기에 대한 Callback 함수. 창이 Resize 됐을 때 호출된다.
typedef void (* GLFWwindowsizefun)(GLFWwindow* window, int width, int height);
GLFWAPI GLFWwindowsizefun glfwSetWindowSizeCallback(GLFWwindow* window, GLFWwindowsizefun callback);

```

Callback 함수가 호출됐을 때 void 반환형의 함수가 실행되게 할 수 있다. 이는 다른 Callback 함수들도 마찬가지이다. 이를 이용해 Callback 함수 호출 시 원하는 이벤트 객체를 생성하고, 어플리케이션의 상위단에서 해당 이벤트를 적절히 처리하게 설계할 수 있다. 아래 과정으로 이벤트가 전파되게 설계했다.

**GLFW Callback 함수 호출 -> Lambda 함수 실행 -> 이벤트 객체 생성 -> 이벤트 전파**

#### Event Class
GLFW Callback 함수에서 생성할 이벤트 객체를 만들기 위해 Event Class를 구현했다. enum class로 이벤트 타입과 카테고리를 정의하고, 구체화된 Event Class에 값을 넣어주었다.

```cpp
enum class EEventType
{
	NONE = 0,
	WINDOW_CLOSE,
	WINDOW_RESIZE,
	WINDOW_FOCUS,
	WINDOW_LOST_FOCUS,
	WINDOW_MOVED, // window
	APP_TICK,
	APP_UPDATE,
	APP_RENDER, // app
	KEY_PRESSED,
	KEY_RELEASED,
	KEY_TYPED, // key
	MOUSE_BUTTON_PRESSED,
	MOUSE_BUTTON_RELEASED,
	MOUSE_MOVE,
	MOUSE_SCROLLED // mouse
};

enum EEventCategory
{
	NONE = 0,
	EVENT_CATEGORY_APP = BIT(0),
	EVENT_CATEGORY_INPUT = BIT(1),
	EVENT_CATEGORY_KEYBOARD = BIT(2),
	EVENT_CATEGORY_MOUSE = BIT(3),
	EVENT_CATEGORY_MOUSE_BUTTON = BIT(4)
};

```

#### 추상화

```cpp
class Event
{
	friend class EventDispatcher;

  public:
	virtual ~Event() = default;
	virtual EEventType getEventType() const = 0;
	virtual const char *getName() const = 0;
	virtual int getCategoryFlags() const = 0;

	virtual std::string toString() const
	{
		return getName();
	}

	inline bool isInCategory(EEventCategory category)
	{
		return getCategoryFlags() & category;
	}

	bool m_Handled = false;
};

```

#### 구체화
Event는 AppEvent, KeyEvent, MouseEvent로 구체화했다. 코드를 다 설명하는 것은 불필요하니 WindowResizeEvent 하나만 예시로 설명하겠다.

``` cpp
class WindowResizeEvent : public Event
{
  public:
	WindowResizeEvent(unsigned int width, unsigned int height) : m_Width(width), m_Height(height)
	{
	}

	unsigned int getWidth() const
	{
		return m_Width;
	}

	unsigned int getHeight() const
	{
		return m_Height;
	}

	std::string toString() const override
	{
		std::stringstream ss;
		ss << "WindowResizeEvent: " << m_Width << ", " << m_Height;
		return ss.str();
	}

	EVENT_CLASS_TYPE(WINDOW_RESIZE);
	EVENT_CLASS_CATEGORY(EVENT_CATEGORY_APP);

  private:
	unsigned int m_Width, m_Height;
};
```

이벤트 타입과 카테고리를 설정하는 코드는 어차피 모든 이벤트 클래스에서 동일하게 진행되기 때문에 매크로를 활용해 반복되는 코드를 줄였다. 
이렇게 밑작업을 하면 이벤트 별로 가지고 있는 특수한 속성만 정의해주면 된다.

``` cpp

#define EVENT_CLASS_TYPE(type)                                                                                         \
	static EEventType getStaticType()                                                                                  \
	{                                                                                                                  \
		return EEventType::type;                                                                                       \
	}                                                                                                                  \
	virtual EEventType getEventType() const override                                                                   \
	{                                                                                                                  \
		return getStaticType();                                                                                        \
	}                                                                                                                  \
	virtual const char *getName() const override                                                                       \
	{                                                                                                                  \
		return #type;                                                                                                  \
	}

#define EVENT_CLASS_CATEGORY(category)                                                                                 \
	virtual int getCategoryFlags() const override                                                                      \
	{                                                                                                                  \
		return category;                                                                                               \
	}

```

#### GLFW Callback 호출, Lambda 함수, 이벤트 객체 생성

``` cpp

// 창에 저장되어 있는 사용자 정의 데이터
glfwSetWindowUserPointer(m_Window, &m_Data);

// Lambda 함수 호출
glfwSetWindowSizeCallback(m_Window, [](GLFWwindow *window, int32_t width, int32_t height) {
    WindowData &data = *(WindowData *)glfwGetWindowUserPointer(window);
    data.width = width;
    data.height = height;

    // 이벤트 객체 생성
    WindowResizeEvent event(width, height);
    data.eventCallback(event);
});
```

이렇게 하면 창이 resize 됐을 때 WindowResizeEvent 객체를 생성하고, eventCallback에 생성된 객체를 매핑할 수 있다. EventCallback을 구현하기 위해 std::function을 활용했다. std::function은 함수 포인터와 다르게 함수를 객체처럼 사용할 수 있게 C++에 도입된 기능이다. 함수, 람다, 멤버 함수 등 Callable한 것을 하나의 타입으로 다룰 수 있다. Window 객체에 std::function<void(Event &)> 형태로 EventCallback을 정의해놓고, 생성된 이벤트 객체를 매개변수로 매핑했다. 

#### 이벤트 전파
이벤트를 생성했다면 내가 원하는 함수가 호출되게 만들어야 한다. 그래서 이벤트를 전파하는 클래스를 정의했다. dispatch 함수는 이벤트 타입 비교 후, 타입이 일치한다면 함수가 실행되게 만들었다.

``` cpp
class EventDispatcher
{
  public:
	EventDispatcher(Event &event) : m_Event(event)
	{
	}

	template <typename T, typename F> bool dispatch(const F &func)
	{
		if (m_Event.getEventType() == T::getStaticType())
		{
			m_Event.m_Handled |= func(static_cast<T &>(m_Event));
			return true;
		}
		return false;
	}

  private:
	Event &m_Event;
};

```

이벤트를 처리하기 위한 모든 구성요소가 만들어졌다. 

EventCallback을 가지고 있는 객체는 Window이지만, Window를 관리하는 객체는 App이다. 엔진의 가장 상위단인 App에서 Window을 생성하고, Window의 EventCallback으로 App의 onEvent 함수를 등록해준다.

```cpp
App::App(...) : ...
{
    // ...
	
    m_Window = std::unique_ptr<Window>(Window::create());
	m_Window->setEventCallback(AL_BIND_EVENT_FN(App::onEvent));

    // ...
}

void App::onEvent(Event &e)
{
	EventDispatcher dispatcher(e);
	dispatcher.dispatch<WindowCloseEvent>(AL_BIND_EVENT_FN(App::onWindowClose));
	dispatcher.dispatch<WindowResizeEvent>(AL_BIND_EVENT_FN(App::onWindowResize));

	// ...
}
```


## 정리

#### 초기화
**App 생성 -> Window 생성 -> GLFW Callback 함수와 Lambda 함수 매핑**

#### 이벤트 전파 과정
 **GLFW Callback 함수 호출 -> Lambda 함수 실행 -> 이벤트 객체 생성 -> -> App::onEvent 실행 -> EventDispatcher가 Event 전파**

글을 적다보니 초기화 과정부터 설명하는 게 뒤의 과정을 이해하는 데 훨씬 수월했을 것 같다. 하지만 초기화 과정부터 설명하려면 App이 어떻게 초기화되고 그 과정에서 또 어떤 일이 일어나는지 설명하는 포스트를 작성해야 할 것 같았기 때문에 이 정도로 끝내겠다.

 Event System을 구현하는 과정에서 std::function, Lambda function, std::bind, std::forward 등 C++의 여러 기능들에 대해 배울 수 있어 좋았다. 이를 어떻게 하면 잘 활용할 수 있을지에 대한 포스트는 나중에 따로 적어보려고 한다. 

 또 배운 점은 확장성 있는 코드를 작성하려면 추상화 - 구체화 과정이 필수적이라는 점이다. 이를 활용하면 모든 OS, Graphics API를 지원하는 코드를 작성할 수 있을 것이다. 

다음에는 엔진의 핵심인 ECS에 관한 포스트로 돌아오겠다!
