---
title: Http Request & Response handling in Webserv + Config
tags: [Network, Http]
style: fill
color: primary
description: Webserv(nginx) Project의 핵심 구현부에 대한 설명.
---

## 도입
지난 포스트에서 kqueue를 사용해 I/O Multiplexing을 어떻게 구현했는지 설명했다. 오늘은 서버 소켓을 만들기 전에 서버 설정 단계를 의마하는 Configuration 파일에 대한 것, 그리고 클라이언트의 Request에 따라 서버가 어떻게 Response를 생성해야 하는지에 대한 포스트를 작성할 것이다.

## Configuration file
Configuration 파일은 서버의 포트 번호, 이름, URL 규칙 등 웹 서버의 설정에 관한 정보가 담겨있는 파일이다. 이렇게만 설명하면 잘 와 닿지 않을 수 있으니, 우리의 config 파일을 예시로 들어 설명해보겠다.

``` json

SOFTWARE_NAME 		nginx;
SOFTWARE_VERSION 	0.1;

HTTP_VERSION		1.1;
CGI_VERSION			1.1;

server {
	listen			80;
	server_name		localhost;

	error_page			404				./usr/html/error.html;

	#file upload path
	filepath	./usr/html/upload;

	location / {
		limit_except		GET POST DELETE;
		# return			301 http://naver.com;
		cgi					php:./usr/cgi/upload.php  bla:./usr/cgi/upload.bla py:./usr/cgi/get_script.py;
		client_body_size 	110000000;
		root				./usr/html;
		index				index.html;
		autoindex			ON;
	}
	location /cgi {
		cgi					php:./usr/cgi/upload.php  bla:./usr/cgi/cgi_tester py:./usr/cgi/get_script.py;
		root	./usr/cgi;
	}
}

```

configuration 파일은 전역 설정, server block, location block 의 계층적 구조로 구성된다. 
전역 설정은 과제에서 꼭 필요한 항목은 아니지만, nginx의 configuration 파일을 확인했을 때 비슷한 항목들이 있어 넣어봤다.

#### 전역 설정
- SOFTWARE_NAME: 서버 이름 또는 식별자
- SOFTWARE_VERSION: 서버 버전 정보
- HTTP_VERSION: 지원하는 HTTP 버전
- CGI_VERSION: 지원하는 CGI 프로토콜 버전

#### server block
하나의 서버 단위를 의미

- listen: 서버의 포트 번호
- server_name: 서버에 연결될 도메인 또는 호스트 이름
- error_page: 에러 발생시 반환할 코드 번호와 페이지
- filepath: 클라이언트가 업로드한 파일을 저장할 경로

#### location block
특정 URI 경로에 대한 처리 방식 지정

- limit_except: 해당 location block에서 허용할 HTTP 메서드
- cgi: cgi 처리 시 사용 가능한 확장자와 실행 파일 경로
- root: 해당 경로에서 파일을 찾을 실제 디렉토리 경로
- index: 기본적으로 반환할 파일명
- autoindex: 요청한 경로가 디렉토리일 경우, 파일 목록을 자동으로 보여줄지 여부
- client_body_size:

nginx에서 사용하는 설정은 이것보다 훨씬 많지만, 과제에서 요구하는 설정은 그렇게 많지 않기 때문에 이 정도에서 마치겠다.

## Configuration file Parsing
파싱은 다음과 같은 과정으로 이루어졌다.

-  *.conf 파일 읽기
-  string -> Config class
-  server block 파싱
-  locate block 파싱

파싱 과정 중 생기는 오류를 효과적으로 분류하기 위해 Status라는 클래스를 만들어 관리했다. Webserv 프로젝트는 Google Convention을 사용했는데, 해당 Convention에서 exception을 사용하지 않는다고 명시되어 있어 Status를 도입하게 됐다.

``` c++
#ifndef STATUS_HPP
# define STATUS_HPP

#include <iostream>

class Status
{
public:
	Status();
	static 	Status OK(void);
	static 	Status Error(const std::string& message);
	static	Status create(const std::string& message);
	bool	ok(void) const;
	const	std::string& message(void) const;
private:
	Status(const std::string& message);
	std::string _message;
};

#endif
```

#### conf 파일 읽기
``` c++
Status Config::ReadConfig(std::string& file)
{
	if (!utils::CheckExtension(file, ".conf"))
		return Status::Error("file extension error");
	
	std::ifstream infile(file, std::ios::in | std::ios::binary);
	if (!infile)
		return Status::Error("Open error");
	infile.seekg(0, std::ios::end);
	size_t size = infile.tellg();
	infile.seekg(0, std::ios::beg);
	char *buffer = new char[size + 1];
	memset(buffer, 0, size + 1);
	if (!infile.read(buffer, size))
	{
		infile.close();
		return Status::Error("Read error");
	}
	infile.close();
	std::string file_content(buffer);
	delete []buffer;

	Status status = ParseConfig(file_content);
	if (!status.ok())
		return Status::Error(status.message());
	return Status::OK();
}
```

입력으로 들어온 *.conf 파일을 읽어 string으로 변환한다. 

#### string -> Config class

``` c++
Status Config::ParseConfig(std::string& file)
{
	std::istringstream iss(file);
	std::string str;
	Status status;
	while (getline(iss, str, '\n'))
	{
		if (str.find('#') != std::string::npos || str.empty() || utils::IsStrSpace(str))
			continue;
		if (str.find("server") == std::string::npos && !utils::CheckTerminator(str))
			return Status::Error("Terminator error");
		if (utils::find(str, "SOFTWARE_NAME"))
			status = ParseSoftwareName(str);
		else if (utils::find(str, "SOFTWARE_VERSION"))
			status = ParseSoftwareVer(str);
		else if (utils::find(str, "HTTP_VERSION"))
			status = ParseHttpVer(str);
		else if (utils::find(str, "CGI_VERSION"))
			status = ParseCgiVer(str);
		else if (utils::find(str, "server"))
			status = ParseServerVariable(str, iss);
		else
			return Status::Error("wrong config option error");
		if (!status.ok())
			return Status::Error(status.message());
	}
	if (!CheckConfigOpt())
		return Status::Error("Essential config option not included");
	if (this->server_vec.size() == 0)
		return Status::Error("no server error");
	if (CheckPortDup())
		return Status::Error("port duplicate error");
	return Status::OK();
}
```

std::istringstream을 사용해 '\n'을 기준으로 앞에서 설명한 전역 설정에 해당하는 부분을 클래스의 멤버 변수에 저장했다. 전역 설정은 중복으로 등장하는 부분만 예외로 처리했다. 그 다음, "server"로 시작하는 행을 찾게 되면 server block 파싱을 시작했다.

#### server block 파싱
server block은 "server {}" 에서 중괄호 안의 내용에 해당한다. 따라서 먼저 중괄호 안의 내용만 string으로 추출했다.

``` c++
Status Config::ParseServerVariable(std::string& file, std::istringstream& iss)
{
	std::string server_block = ExtractServerBlock(iss, file);
	if (server_block.empty())
		return Status::Error("server block error");
	Server server;
	Status status = server.ParseServerBlock(server_block);
	if (status.ok())
		this->server_vec.push_back(server);
	return status;
}
```

그 후에는 전역 설정 항목 파싱과 비슷한 맥락으로 server block을 파싱했다.

``` c++
Status Server::ParseServerBlock(std::string& server_block)
{
	std::istringstream iss(server_block);
	std::string str;
	Status status;
	while (getline(iss, str, '\n'))
	{
		if (str.find('#') != std::string::npos || str.empty() || utils::IsStrSpace(str))
			continue;
		if (str.find("location /") == std::string::npos && !utils::CheckTerminator(str))
			return Status::Error("Terminator error");
		else if (str[str.length() - 1] == ';')
			str.resize(str.length() - 1);
		if (utils::find(str, "listen"))
			status = ParsePortVariable(str);
		else if (utils::find(str, "server_name"))
			status = ParseServerName(str);
		else if (utils::find(str, "error_page"))
			status = ParseErrorPage(str);
		else if (utils::find(str, "filepath"))
			status = ParseFilePath(str);
		else if (utils::find(str, "location"))
			status = ParseLocateVariable(str, iss);
		else
			return Status::Error("wrong config option error");
		if (!status.ok())
			return Status::Error(status.message());
	}
	if (!CheckServerOpt())
		return Status::Error("Essential server option not included");
	if (this->locate_vec.size() == 0)
		return Status::Error("no location error");
	return Status::OK();
}
```

예외처리로는 항목이 중복으로 등장하는지 여부, 포트 범위가 0 ~ 65535인지, error code가 300 ~ 599 사이인지, location block이 존재하는지, listen, server_name, filepath가 모두 등장하는지를 확인했다.

#### locate block 파싱
``` c++
Status Locate::ParseLocateBlock(std::string& locate_block)
{
	std::istringstream iss(locate_block);
	std::string str;
	Status status;
	while (getline(iss, str, '\n'))
	{
		if (str.find('#') != std::string::npos || str.empty() || utils::IsStrSpace(str))
			continue;
		if (!utils::CheckTerminator(str))
			return Status::Error("Terminator error");
		else
			str.resize(str.length() - 1);
		if (utils::find(str, "limit_except"))
			status = ParseMethod(str);
		else if (utils::find(str, "return"))
			status = ParseRedirect(str);
		else if (utils::find(str, "root"))
			status = ParseRoot(str);
		else if (utils::find(str, "index"))
			status = ParseIndex(str);
		else if (utils::find(str, "autoindex"))
			status = ParseAutoIndex(str);
		else if (utils::find(str, "cgi"))
			status = ParseCgi(str);
		else if (utils::find(str, "client_body_size"))
			status = ParseClientSize(str);
		else
			return Status::Error("wrong config option error");
		if (!status.ok())
			return Status::Error("Parsing error");
	}
	if (!CheckLocateOpt())
		return Status::Error("Essential location option not included");
	return Status::OK();
}
```

앞에서 server block을 string으로 추출했던 것처럼, locate block을 string으로 추출한 후 파싱을 진행했다. 예외 처리로는 허용 메서드 중 post, get, delete을 제외한 다른 메서드가 있는지, client_body_size가 허용 범위를 넘어서는지, root가 등장하는지, 중복으로 등장하는 항목이 없는지를 확인하여 처리했다.

이렇게 파싱을 마치고 나면, Config라는 클래스 안에 필요한 정보가 모두 들어있게 된다. 이후에는 ServerManager에서 Config의 Server block 정보를 바탕으로 서버 소켓을 생성하게 된다. 그리고 클라이언트의 request를 location block의 정보를 바탕으로 처리할 수 있게 된다.

## Http Message
클라이언트의 request, 서버의 response를 설명하기 전에 Http 메시지에 대해 먼저 설명하고 넘어가겠다. 
Http 메시지는 단순한, 데이터의 구조화된 블록이다. 메시지는 시작줄, 헤더 블록, 본문 이렇게 세 부분으로 이루어진다. 시작줄은 이것이 어떤 메시지인지 서술하며, 헤더 블록은 속성을, 본문은 데이터를 담고 있다. 

![http message](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdna%2FceqzRL%2FbtrFHEJFUZb%2FAAAAAAAAAAAAAAAAAAAAANG-cLHTNU3CTXaw5CPcQT7NThXHpO43yQwP64pC5N57%2Fimg.png%3Fcredential%3DyqXZFxpELC7KVnFOS48ylbz2pIh7yKj8%26expires%3D1753973999%26allow_ip%3D%26allow_referer%3D%26signature%3DBQJrez2SPYo7sYI1tg1ouAn2ii0%253D)

<sub>이미지 출처: [https://hahahoho5915.tistory.com/62)</sub>

시작줄과 헤더는 줄 단위로 분리된 아스키 문자열이다. 각 줄은 /r/n 으로 끝나고, 이 줄바꿈 문자열을 'CRLF'라고 부른다. 본문은 단순한 데이터 덩어리이다. 시작줄이나 헤더와는 달리, 텍스트나 binary 데이터를 포함할 수도 있고, 비어있을 수도 있다. 

Http 메시지는 요청 메시지와 응답 메시지로 분류 된다.

#### 요청 메시지

```
<메서드> <요청 URL> <버전>
<헤더>

<본문>
```

#### 응답 메시지

```
<버전> <상태 코드> <사유 구절>
<헤더>

<본문>
```

각 부분에 대한 설명은 다음과 같다.

- 메서드: 클라이언트 측에서 서버가 리소스에 대해 수행해주길 바라는 동작. 'GET', 'POST', 'DELETE' 등과 같이 한 단어로 되어 있다. 
- 요청 URL: 요청 대상이 되는 리소스를 지칭하는 완전한 URL 혹은 URL의 경로 구성요소이다. 
- 버전: 메시지에서 사용 중인 Http 버전이다. 
- 상태 코드: 요청 중에 무엇이 일어났는지 설명하는 세 자리 숫자이다.  
