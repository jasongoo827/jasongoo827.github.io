---
title: I/O Multiplexing in Webserv
tags: [Network, Multiplexing, kqueue, Server]
style: fill
color: primary
description: Webserv(nginx) Project의 I/O Multiplexing에 대한 이론부터 구현까지에 대한 자세한 설명.
---

## I/O 작업
I/O Multiplexing에 대해 설명하기에 앞서, 먼저 I/O 작업이 무엇인지 이해할 필요가 있다. I/O는 Input/Ouput의 약자로, 입력과 출력을 의미한다. 여기서 입력이란 외부의 데이터를 받아오는 작업이며, 출력은 프로그램이 처리한 데이터를 외부로 내보내는 작업이다.
네트워크 프로그래밍에서 I/O작업은 주로 socket을 통한 데이터 송수신을 의미한다. Linux/Unix 환경에서는 socket 또한 파일 즉, 파일 디스크립터(File Descriptor, FD)로 관리되기 때문에, 좀 더 넓은 관점에서 보면 이는 곧 서버와 클라이언트 간의 통신을 포함한 파일 입출력 처리 방식 전반에 대한 이야기라고 할 수 있다.

결국 웹서버와 같은 네트워크 기반 애플리케이션에서 I/O 작업이란, **"언제 어떤 방식으로 데이터를 주고받을 것인가?"**를 정의하는 핵심적인 방식이라 할 수 있다.

## 기본개념
앞서 언급했듯이, 네트워크 프로그램에서 I/O 작업은 데이터를 읽거나 쓰는 과정이다. I/O 작업은 user space에서 직접 수행할 수 없기 때문에, user process가 kernel에 I/O 작업을 요청하고 응답받는 구조이다.
응답 받는 순서(Synchronous/Asynchronous), 받는 타이밍(Blocking/Non-Blocking)에 따라 여러 모델로 분류할 수 있다.

⸻
### Synchronous
- 모든 I/O 작업이 일련의 순서를 가지고 진행된다.
- 작업 완료를 user space에서 판단해, 다음 작업을 언제 요청할지 결정한다.

### Asynchronous
- kernel에 I/O 작업을 요청하고 다른 작업을 할 수 있으나, 순서가 보장되지는 않는다.
- 작업 완료를 kernel space에서 통보해 준다.


![Sync&Async](https://evan-moon.github.io/static/e075bee6dfcc71a33568cd0ef7b6f61c/dc0d9/thumbnail.webp)
<sub>이미지 출처: [https://evan-moon.github.io/2019/09/19/sync-async-blocking-non-blocking/](https://www.geeksforgeeks.org/http-full-form/)</sub>

⸻

### Blocking
- 요청한 작업이 끝날 때까지 기다렸다가 끝나면 그 결과를 돌려받는다.

### Non-Blocking
- 작업 요청 이후 결과는 나중에 필요할 때 전달 받는다.
- 중간중간 상태를 확인해 볼 수는 있다.


![Block-NonBlock](https://mark-kim.blog/static/9172fd37743404e3409996057cb8b526/17e4a/synchronous_vs_asynchronous.webp)

<sub>이미지 출처: [https://mark-kim.blog/blocking_nonblocking_&_synchronous_asynchronous/](https://www.geeksforgeeks.org/http-full-form/)</sub>

⸻

이 4가지 기준에 따라 I/O 모델을 4가지로 분류할 수 있는데, 각 모델들에 대한 설명을 잘 해놓은 글이 많기 때문에 자세한 설명을 하지는 않겠다.   

## I/O Multiplexing - Asynchronous Blocking
웹서버와 같은 네트워크 프로그램은 수십, 수백 개의 클라이언트와 동시에 통신해야 하는 상황에 자주 놓인다.
단순한 Blocking 방식으로는 하나의 클라이언트 요청을 처리하는 동안 다른 클라이언트는 응답을 받을 수 없다. 이를 해결하기 위해 I/O Multiplexing이라는 개념이 등장했다.

I/O Multiplexing은 하나의 프로세스가 여러 개의 파일 디스크립터(FD)를 동시에 감시하고, 데이터를 읽거나 쓸 수 있는 상태가 되었을 때만 처리하는 방식이다.
수많은 파일 디스크립터 중 현재 작업 가능한 것만 골라서 처리하기 때문에, CPU 자원을 효율적으로 사용하며 다수의 연결을 관리할 수 있다.

운영체제마다 제공하는 Multiplexing 방식이 약간씩 다른데, 몇 가지 종류에 대해 설명하겠다.

### select
- 가장 오래된 I/O Multiplexing 방식
- 고정된 크기의 FD 배열을 사용(1024개)
- 모든 FD를 매번 순회하여 체크
- 모든 OS에서 지원

``` c
fd_set readfds;
select(max_fd + 1, &readfds, NULL, NULL, NULL);
```

### poll
- select의 한계를 개선한 방식 (FD 수 제한 없음)
- 배열 기반이지만 순회 필요
- 각 FD마다 구조체 pollfd를 사용
- 이벤트를 감지하지만, 어떤 FD인지 매번 순회해야 함

``` c++
struct pollfd fds[NUM];
poll(fds, NUM, TIMEOUT);
```

### epoll
- Linux에서 select/poll의 단점을 극복한 고성능 API
- 커널이 이벤트 대기 목록을 관리
- 이벤트 기반 -> 변경된 FD만 알려줌
- epoll_create, epoll_ctl, epoll_wait로 구성

``` c++
int epfd = epoll_create(1);
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event);
epoll_wait(epfd, events, MAX_EVENTS, -1);
```

### IOCP
- Windows에서 제공하는 Asynchronous I/O 모델
- 커널이 I/O 완료를 비동기로 알려주는 완전한 비동기 모델
- 스레드풀과 연계되어 고성능 멀티 I/O 처리 가능

```c++
HANDLE iocp = CreateIoCompletionPort(...);
GetQueuedCompletionStatus(iocp, ...);
```

### kqueue
- macOS, FreeBSD, NetBSD 등에서 사용되는 고성능 이벤트 큐
- epoll과 유사하게 변경된 이벤트만 감지
- 이벤트 등록과 감지를 모두 kevent()로 처리

Webserv 과제는 macOS에서 진행해야 하기 때문에, kqueue를 사용해서 I/O Multiplexing을 구현했다. 
kqueue를 사용하기 위해 알아야 하는 것이 몇 가지 있다.

- kqueue(), kevent()

```c++
#include <sys/event.h>

int kqueue();

int kevent(int kq,
    const struct kevent * changelist, int nchanges,
    struct kevent * eventlist, int nevents,
    const struct timespec * timeout);
```

kqueue() 함수는 새로운 이벤트 큐를 생성하는 함수이다. 반환값은 이벤트 큐의 File Descriptor이다. 이는 이후에 kevent() 호출에 사용된다.

kevent() 함수는 이벤트를 등록하거나, 발생한 이벤트를 감지할 때 사용하는 시스템 콜이다. 이벤트 등록할 때는 changelist에 담아 넘기면 되고, 감지하려면 eventlist에 발생한 이벤트들을 담아 반환받는다.
timeout을 통해 대기 시간 조정도 가능하다.


- kevent 구조체

```c++
struct kevent {
    uintptr_t  ident;    // 감시 대상 (파일 디스크립터 등)
    int16_t    filter;   // 이벤트 종류 (ex: EVFILT_READ, EVFILT_WRITE)
    uint16_t   flags;    // 동작 설정 플래그 (ex: EV_ADD, EV_ENABLE)
    uint32_t   fflags;   // filter-specific 플래그 (많이 사용되진 않음)
    intptr_t   data;     // 이벤트에 따라 의미가 달라짐 (ex: read 가능 바이트 수)
    void*      udata;    // 유저 정의 포인터 (컨텍스트 전달용)
};
```

kevent는 이벤트를 등록하거나, 이벤트 발생 시 정보를 전달받을 때 사용하는 구조체이다. ident, filter, flag의 값을 통해 이벤트에 대한 정보를 알 수 있다.

EV_SET 매크로를 사용해 이벤트를 등록할 수 있다.

``` c++

struct kevent change;
EV_SET(&change, client_fd, EVFILT_READ, EV_ADD | EV_ENABLE, 0, 0, NULL);
kevent(kq, &change, 1, NULL, 0, NULL);

```

⸻

## 구현
kqueue를 사용해 서버를 구현하려면 다음과 같은 순서로 구현해야 한다.

1. kqueue 생성
2. Server Socket 생성
3. Server Socket을 kqueue에 등록
4. kqueue에서 발생하는 이벤트 감시
5. 서버가 끊어지면 모든 자원 정리

⸻

#### kqueue 생성

```c++
bool	ServerManager::InitKqueue(int &kq, int &sock_serv)
{
	kq = kqueue();
	if (kq == -1)
	{
		std::cerr << "kqueue fail\n";
		close(sock_serv);
		return (false);
	}
	return (true);
}
```

kqueue 함수를 사용해 커널로부터 이벤트 큐를 생성한다. 

⸻

#### Server Socket 생성

``` c++

bool	ServerManager::InitServerAddress(int &kq, sockaddr_in &addr_serv, int port)
{
	void	*error = memset(&addr_serv, 0, sizeof(addr_serv));
	addr_serv.sin_family = AF_INET;
	addr_serv.sin_port = htons(port);
	addr_serv.sin_addr.s_addr = htonl(INADDR_ANY);
	if (error == NULL)
	{
		// ...
	}
	return (true);
}

```

서버 소켓을 만들기 전에 먼저 sockaddr_in 구조체를 사용해 포트와 IP 주소를 설정해준다. 

AF_INET은 IPv4를 의미하고, IPv6를 쓰려면 AF_INET6을 사용한다. 여기서는 IPv4를 사용하니 AF_INET으로 설정한다. 

htons(host-to-network short) 호스트 바이트 순서의 IP 포트 번호를 네트워크 바이트 순서의 IP 포트 번호로 변환한다. 

htonl(host-to-network long) 함수를 사용하여 호스트 바이트 순서의 IPv4 주소를 네트워크 바이트 순서의 IPv4 주소로 변환한다. INADDR_ANY는 0.0.0.0을 의미하며, 모든 IP 주소로 들어오는 요청을 받겠다는 뜻이다.


``` c++
bool	ServerManager::InitServerSocket(int &kq, int &sock_serv, sockaddr_in &addr_serv)
{
	sock_serv = socket(AF_INET, SOCK_STREAM, 0);
	if (sock_serv == -1)
	{
		// 예외 처리
	}

	int enable = 1;
	if (setsockopt(sock_serv, SOL_SOCKET, SO_REUSEADDR, &enable, sizeof(int)) < 0)
	{
		// ...
	}

	if (bind(sock_serv, (struct sockaddr*)&addr_serv, sizeof(addr_serv)) == -1)
	{
		// ...
	}

	if (listen(sock_serv, 2048) == -1)
	{
		// ...
	}

	if (utils::SetNonBlock(sock_serv) == false)
	{
		// ...
	}
	return (true);
}

```

socket() 함수를 사용해 서버 소켓을 만든다. AF_INET은 IPv4, SOCK_STREAM은 TCP 소켓 형식을 의미한다. 3번째 매개 변수는 프로토콜을 의미하는데, 0을 값으로 넣으면 자동으로 첫 번째, 두 번째 매개변수를 기준으로 인자값을 지정해준다.

setsockopt() 함수를 사용해 socket 옵션을 설정해준다. SOL_SOCKET은 옵션을 socket level로 설정하는 것이다. SO_REUSEADDR은 소켓이 다른 소켓에서 사용 중인 포트에 강제로 바인딩할 수 있게 하는 옵션인데, 서버를 자주 껐다 켰다 하는 경우에 유용하다.

bind() 함수를 사용해 앞에서 만든 소켓과 IP주소-포트 를 묶어준다.

이제 listen() 함수를 통해 바인딩이 끝난 서버 소켓을 클라이언트의 요청을 받을 수 있는 상태로 만든다. 두 번째 인자인 2048은 백로그 큐의 크기로, 동시에 대기할 수 있는 연결 요청의 수를 의미한다.

마지막으로 fcntl() 함수를 사용해 서버 소켓, 즉 파일 디스크립터를 O_NONBLOCK 설정 해준다. read 시 blocking으로 처리하면 데이터를 읽어올 때까지 기다리기 때문에 non blocking으로 설정해줘야 한다.

⸻

#### Server Socket을 kqueue에 등록

``` c++

bool	ServerManager::RegistSockserv(int &kq, int &sock_serv, struct ::kevent &change_event)
{
	EV_SET(&change_event, sock_serv, EVFILT_READ, EV_ADD | EV_ENABLE, 0, 0, NULL);
	if (::kevent(kq, &change_event, 1, NULL, 0, NULL) == -1)
	{
		// ...
	}
	return (true);
}

```

서버 소켓을 생성하고 바인딩한 후에, 클라이언트의 요청을 감지하기 위해 kqueue에 해당 소켓을 읽기 이벤트 대상으로 등록해야 한다. 

⸻

#### kqueue에서 발생하는 이벤트 감시

``` c++

bool	ServerManager::CheckEvent(int &kq, struct ::kevent *events, int &event_count)
{
	event_count = kevent(kq, NULL, 0, events, 60, NULL);
	if (event_count == -1)
	{
		std::cerr << "kevent wait fail\n";
		return (false);
	}
	return (true);
}

```

kevent 함수를 사용해 event_count, events 배열을 받아온다.

앞에서 만든 kqueue와 서버 소켓을 사용해 events 배열을 순회해, 클라이언트에서 들어온 요청을 처리하고, 발생한 이벤트들을 처리해야 한다. 이벤트 종류를 크게 3가지로 분류할 수 있다.

- 서버 소켓으로 읽기 이벤트 발생 

``` c++
if (events[i].filter == EVFILT_READ && CheckValidServer(events[i].ident))
{
    int 				sock_client;
    struct sockaddr_in	addr_client;
    if (InitClientSocket(kq, sock_serv, change_event, sock_client, addr_client, sizeof(addr_client)) == false)
        continue ;
    Connection *con = new Connection(kq, sock_client, addr_client, config, &session);
    v_connection.push_back(con);
    AddConnectionMap(sock_client, v_connection.back());
}

```

서버 소켓으로 읽기 이벤트가 발생했다면 클라이언트에서 새로운 연결을 요청한 것이기 때문에, accept() 함수를 통해 클라이언트 소켓을 생성하고 앞에서 서버 소켓의 옵션을 설정해줬던 것처럼 동일하게 진행 후, kquque에 등록해준다. 클라이언트 소켓 생성 시 Connection이라는 객체를 만들어, 클라이언트의 연결과 관련된 정보를 관리한다. 이를 std::map에 <fd, Connection*>로 저장해 request, response를 처리할 때 사용했다.

- 클라이언트 소켓 이벤트 발생

``` c++
// if Client Socket Event
{
    if (connectionmap.find(static_cast<int>(events[i].ident)) == connectionmap.end())
        continue;
    Connection* connection = connectionmap[static_cast<int>(events[i].ident)];
    connection->MainProcess(events[i]);
    connection->UpdateTimeval();
    AfterProcess(connection);
}

```

클라이언트 소켓 이벤트 발생 시, request에 대한 적절한 response를 만들어 클라이언트에게 보낸다. 이에 관한 부분은 따로 자세히 포스트로 다룰 예정이다.


- 나머지 경우

이미 클라이언트의 요청을 처리했지만, 알 수 없는 이유로 계속 요청이 들어오는 경우가 있었다. 이 경우에 EV_EOF 플래그가 켜졌었는데, 이 경우에 연결을 정상적으로 끊을 수 있게 처리해주었다. 나중에 알고보니 Connection을 keep-alive로 구현하지 않아 생긴 문제였는데, keep-alive로 수정 후에는 EV_EOF 플래그가 켜지지 않았다. 코드는 따로 첨부하지 않겠다.

⸻

#### 서버가 끊어지면 모든 자원 정리

``` c++
void	ServerManager::CloseAllConnection()
{
	for (size_t i = 0; i < v_connection.size(); ++i)
	{
		if (v_connection[i]->GetFileFd())
		{
			CloseConnectionMap(v_connection[i]->GetFileFd());
			v_connection[i]->SetFileFd(0);
		}
		else if (v_connection[i]->GetPipein())
		{
			CloseConnectionMap(v_connection[i]->GetPipein());
			v_connection[i]->SetPipein(0);
		}
		else if (v_connection[i]->GetPipeout())
		{
			CloseConnectionMap(v_connection[i]->GetPipeout());
			v_connection[i]->SetPipeout(0);
		}
		CloseVConnection(v_connection[i]->GetClientSocketFd());
	}
	close(kq);
	v_connection.clear();
	connectionmap.clear();
}
```

서버가 종료됐을 때 보관하고 있는 Connection, 연관된 자원들을 모두 정리하고 프로그램을 종료한다.

## 마무리
지금까지 kqueue를 활용한 I/O Multiplexing 방식의 서버 구현 흐름을 하나씩 살펴보았다.
단순히 동작하는 서버를 만드는 것을 넘어, 운영체제의 이벤트 기반 시스템 콜이 어떻게 작동하는지 이해하고, 소켓을 어떻게 효율적으로 관리할 수 있는지 체계적으로 정리해보았다.

다음 포스트에서는 이 구조를 기반으로 실제 클라이언트의 요청(Request)을 분석하고, 적절한 응답(Response)을 생성해주는 HTTP 처리 로직에 대해 이야기해보려고 한다. 또한 설정 파일을 기반으로 어떻게 서버가 생성되는지에 대해 설명할 것이다.