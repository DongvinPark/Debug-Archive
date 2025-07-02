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
- Async & Non-blocking
  - 결과물을 기다리지 않으며, 스레드 또한 멈춰 있지 않습니다. Node.js, Libuv, Boost Asio IoContext 등의 구현 예시가 존재합니다.
<br><br/><br><br/>



## Exceptional Control Flow & Computer H/W

### 역할과 기능
- 중개자 역할
  - 사용자 레벨의 프로그램과 시스템 레벨 프로그램(커널) 간의 연결을 담당합니다.
- computer system에서 다음과 같은 기능을 담당합니다.
  - System call entry/exit
  - Signal delivery + handling
  - Virtual memory (page faults, protection faults)
  - Process/thread context switching
  - Handling of hardware I/O (interrupts, keyboard and mouse input)
  - Language-level(S/W level) exception handling (e.g., C++ try/catch)
  - Asynchronous I/O readiness (e.g., select, epoll waiting on interrupts)
- ECF는 이와 같은 중요한 기능들을 담당하고 있기 때문에, ECF를 전혀 사용하지 않는 응용프로그램을 찾아보는 것은 매우 어려울 것입니다.
<br><br/>

### ECF와 Async - Nonblocking I/O
- 내가 직접 그린 그림이 필요하다.
- 여러 명의 클라이언트들이 들어 왔을 때, 컴퓨터 내부에서 무슨 일이 일어나는지를 설명하는 그림과 단계별 설명이 필요하다.
  - 하나의 메인 스레드와 하나의 워커 스레드를 갖추고 요청에 대하여 텍스트 파일 리딩 결과를 write하는 서버가 존재한다.
  - 해당 서버는 2 core CPU 를 갖춘 머신에서 작동하고 있다.
  - 메인 스레드는 select(또는 eqoll) 함수를 주기적으로 호출하면서 요청을 받아들이고, 요청이 올 때마다 파일리딩 작업은 워커 스레드에게 맡긴다.
  - 나머지 하나의 스레드는 메인 스레드의 파일 리딩 요청을 받으면 파일 리딩 작업을 진행하며, 파일 리딩이 완료되면 메인 스레드에게 이벤트 완료를 알린다. 파일 리딩에 얼마의 시간이 걸릴지는 실제로 작업을 해보기 전에는 알 수 없으며 짧을 수도 있고 길수도 있다. 리딩 작업은 실패하지 않는다고 가정한다.
  - 메인 스레드는 파일 리딩이 완료됐다는 이벤트를 받으면 해당 요청을 준 클라이언트의 소켓을 찾아내서 파일 리딩 결과를 write 한다.
  - 이제 이 서버에 두 명의 클라이언트가 거의 동시에 요청을 해 왔고, 먼저 요청한 클랑이언트의 파일 리딩 작업은 오래 결리며 조금 나중에 요청한 클라이언트는 비교적 짧은 시간에 완료될 수 있다고 가정하자.
  - 이러한 상황에서 컴퓨터 내부에서는 무슨 일이 일어날 것인가?
  - 일단 서버가 작동 중이다. 메인 스레드는 select 함수의 블록킹 기능에 의해서 블록 돼 있는 상태로 클라이언트를 대기하는 중이고, 워커 스레드는 아무 일도 하지 않는 상태다.
  - 이때 클라이언트 두 명이 거의 동시에 접속한다. NIC에 의해서 이러한 접속이 감지되고, DMA에 의해서 nic 버퍼가 소켓 버퍼에 복사 된다. NIC는 이러한 이벤트를 CPU에 전달한다.
  - 서버를 돌리고 있던 CPU는 클라이언트 접속 이벤트를 확인하고, ECF 중 인터럽트를 실행한다. select 함수가 리턴하면서 클라이언트 두 명에게 할당된 파일 디스크립터를 응용 프로그램(즉, 서버)에서 접근할 수 있게 된다. 그 결과, 소켓 버퍼의 내용이 7 계층 버퍼에 복사되면서 응용 프로그램은 클라이언트의 요청을 해석할 수 있게 된다.
  - 메인 스레드는 요청에 대응하기 위한 파일 리딩 작업을 자기가 처리하지 않고 워커 스레드에게 맡긴 채, 파일 리딩 함수 호출의 결과물을 기다리지 않고 다음 요청 루프를 기다린다. 이와 동시에 CPU의 다른 코어에서 워커 스레드를 작동시키기 위한 ECF를 처리하여 워커 스레드가 파일 리딩 작업을 시작하게 만든다. 메인 스레드는 여전히 다음 요청을 위해 대기하면서 블록킹된 상태를 유지한다.
  - 워커 스레드는 두 클라이언트를 위한 파일 리딩 작업을 시작하고, 파일 리딩은 OS 커널을 사용해야 하므로 이 또한 ECF로 처리된다. 파일 리딩이 끝나면 워커 스레드는 해당 이벤트를 메인 스레드에 알린다.
  - 메인 스레드는 파일 리딩 완료 이벤트를 수신하고, 파일 리딩 결과를 기다라고 있는 클라이언트 커넥션(즉, 파일 디스크립션이자 소켓)에게 응답을 wirite 한다. 이 또한 OS 커널의 기능을 사용해야 하기 때문에 ECF로 처리 된다.
<br><br/>

### Async - Nonblocking I/O 가 Sync - Blocking 보다 왜 효과적인가?
- '효과적'이라는 말에 어떤 기준으로 봤을 때 '효과적'인지를 잘 정의해야 될 듯 하다.
<br><br/><br><br/>



## 언제 어떻게 사용해야 하는가?
### Sync - Blocking I/O
- 클라이언트 마다 오랫동안 컴퓨팅 리소스를 점유해야 할 때 적합합니다. 결과적으로 클라이언트 1 명당 1 개의 스레드를 할당하게 됩니다.
- connection 별로 state를 유지 및 관리해야 하는 스트리밍 서버의 트래픽이 이러한 양상을 보입니다. RTSP 서버가 대표적입니다.
- 실제 테스트 결과도 존재한다.(내꺼 보여주기)
<br><br/>

### Async - Nonblocking I/O
- 클라이언트 1 명의 컴퓨팅 리소스 점유 시간이 짧고 랜덤할 때 적합합니다. 결과적으로 연결돼 있는 connection(즉, 소켓)은 많지만, 실제로 컴퓨팅 리소스를 장기간 점유하는 connection은 적은 경우에 적합합니다.
- 웹 서버, HTTP 서버의 트래픽이 대표적으로 이러한 양상을 보입니다.
<br><br/><br><br/>



## 참고자료

- Computer Systems A Programmers's Perspective 3rd edition(2016)
  - Randal E. Bryant, David R. O'Hallaron(
    - 199 ~ 386p : Machine-Level Representation of Programs,
    - 757 ~ 835p : Exceptional Control Flow,
    - 953 ~ 1005p : Network Programming,
    - 1007 ~ 1076p : Concurrent Programming)
- 내 (C++ RTSP Streaming Server) 테스트 결과