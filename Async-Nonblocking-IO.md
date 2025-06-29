# H/W 수준에서 파악해본 Async & Non-blocking I/O



## 2 X 2 Matrix - Sync, Async, Blocking, Non-blocking

### 설명 대상의 차이
- {Sync/Async}와 {Blocking/Non-blocking}은 초점을 맞추고 있는 대상이 다릅니다.
- Sync & Async는 프로그램의 일부 control flow에 초점을 맞춥니다. 즉, 리턴 값 또는 결과물을 기다리느냐 그렇지 않느냐입니다.
- Blocking & Non-blocking은 현재의 프로그램을 실행시키는 스레드의 행태에 초점을 맞춥니다. 즉, 스레드가 멈춰 있느냐 그렇지 않느냐입니다.
<br><br/>

### 2 X 2 매트릭스 해석하기
- Sync & Blocking
  - 결과물을 기다릴 때마다 스레드도 멈춰 있습니다. 전통적 파일 리딩, 네트워크 I/O가 이 방식을 택하고 있습니다.
- Sync & Non-blocking
  - 결과물을 기다리지만, 스레드는 멈춰 있지 않습니다. 함수의 호출이 주기적으로 반복됩니다.
- Async & Blocking
  - 결과물을 기다리지 않는데 스레드는 멈춰 있습니다. 실제 구현 예시를 찾아보기 어려우며, 구현 필요성이 적습니다.
- Async & Nonblocking
  - 결과물을 기다리지 않으며, 스레드 또한 멈춰 있지 않습니다. Node.js, Libuv, Boost Asio IoContext 등의 구현 예시가 존재합니다.
<br><br/><br><br/>



## Exceptional Control Flow & Computer H/W

### 역할과 기능
- Bridge 역할
  - 사용자 레벨의 프로그램과 시스템 레벨 프로그램(커널) 간의 연결을 담당합니다.
- computer system에서 다음과 같은 기능을 담당합니다.
  - System call entry/exit
  - Signal delivery + handling
  - Virtual memory (page faults, protection faults)
  - Process/thread context switching
  - Handling of hardware I/O (interrupts, keyboard and mouse input)
  - Language-level(S/W level) exception handling (e.g., C++ try/catch)
  - Asynchronous I/O readiness (e.g., select, epoll waiting on interrupts)
<br><br/>

### 프로그램의 실행과 스레드
- Stack Frame
  - CS:APP에서 유명한 그림첨부한다.
  - 스택 프레임은 각 스레드마다 주어진다.
  - 스택 프레임 안에는 사용자 레벨 함수도 있지만, ECF를 필요로 하는 시스템 레벨 함수도 포함될 수 있다.
<br><br/>

### ECF와 Async - Nonblocking I/O
- 내가 직접 그린 그림이 필요하다.
- 여러 명의 클라이언트들이 들어 왔을 때, 컴퓨터 내부에서 무슨 일이 일어나는지를 설명하는 그림과 단계별 설명이 필요하다.
<br><br/><br><br/>



## 언제 어떻게 사용해야 하는가?
### Sync - Blocking I/O
- 클라이언트 마다 오랫동안 컴퓨팅 리소스를 점유해야 할 때 적합합니다. 결과적으로 클라이언트 1 명당 1 개의 스레드를 할당하게 됩니다.
- connection 별로 state를 유지 및 관리해야 하는 스트리밍 서버의 트래픽이 이러한 양상을 보입니다.
- 실제 테스트 결과도 존재한다.(내꺼 보여주기)
<br><br/>

### Async - Nonblocking I/O
- 클라이언트 1 명의 컴퓨팅 리소스 점유 시간이 짧고 랜덤할 때 적합합니다. 결과적으로 연결돼 있는 connection은 많지만, 실제로 컴퓨팅 리소스를 장기간 점유하는 connection은 적은 경우에 적합합니다.
- 웹 서버, HTTP 서버의 트래픽이 대표적으로 이러한 양상을 보입니다.
<br><br/><br><br/>



## 참고자료

- Computer Systems A Programmers's Perspective 3rd edition(2016)
  - Randal E. Bryant, David R. O'Hallaron(757 ~ 835p, 1007 ~ 1076p)
- 내 (C++ RTSP Streaming Server) 테스트 결과