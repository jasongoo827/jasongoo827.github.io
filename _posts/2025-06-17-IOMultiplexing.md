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

## I/O Multiplexing - Asynchronous Blocking I/O
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
전체적인 코드는 다음과 같다.

## 구현

```c++

bool		ServerManager::RunServer(Config* config)
{
	struct kevent				change_event;
	struct kevent				events[60];

	while(true)
	{
		if (config->GetServerVec().empty())
		{
			std::cerr << "No server configuration\n";
			return false;
		}
		if (InitKqueue(kq, sock_serv) == false)
			continue ;
		
		// Server 초기화
		for (std::vector<Server>::const_iterator it = config->GetServerVec().begin(); it != config->GetServerVec().end(); ++it)
		{
			if (InitServerAddress(kq, addr_serv, it->GetPort()) == false)
			{
				CloseAllServsock();
				break ;
			}
			if (InitServerSocket(kq, sock_serv, addr_serv) == false)
			{
				CloseAllServsock();
				break ;
			}
			if (RegistSockserv(kq, sock_serv, change_event) == false)
			{
				CloseAllServsock();
				break ;
			}
			v_sock_serv.push_back(sock_serv);
		}
		if (kq == 0)
			continue ;
		while (1)
		{
			if (CheckEvent(kq, events, event_count) == false)
				break ;
			for (int i = 0; i < event_count; ++i)
			{
			    // Client Socket 등록 & Connection 추가
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
				// Request or Response 처리
				else if (!(events[i].flags & EV_EOF) || (connectionmap.find(static_cast<int>(events[i].ident)) != connectionmap.end() && (events[i].filter == EVFILT_READ && connectionmap[static_cast<int>(events[i].ident)]->GetProgress() == CGI)))
				{
					if (connectionmap.find(static_cast<int>(events[i].ident)) == connectionmap.end())
						continue;
					Connection* connection = connectionmap[static_cast<int>(events[i].ident)];
					connection->MainProcess(events[i]);
					connection->UpdateTimeval();
					AfterProcess(connection);
				}
				// Connection 끊기
				else if (events[i].flags & EV_EOF)
				{
					// ...
				}
			}
			CheckConnectionTimeout();
		}
		CloseAllConnection();
	}
	return (0);
}

```