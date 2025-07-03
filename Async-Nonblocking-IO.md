# H/W 수준에서 파악해본 Async & Non-blocking I/O



## 2 X 2 Matrix - Sync, Async, Blocking, Non-blocking
|                 | Blocking I/O                                               | Non-blocking I/O                                              |
|-----------------|-----------------------------------------------------------|---------------------------------------------------------------|
| **Synchronous I/O** | - 스레드가 I/O 완료 시까지 멈춤<br>- ex: `read()`, `write()` (일반) | - 스레드가 I/O 시도 → 준비 안 됐으면 실패(EAGAIN)<br>- 루프나 poll로 재시도<br>- ex: `read()` + O_NONBLOCK |
| **Asynchronous I/O** | - 의미상 거의 없음 (async인데 스레드가 멈춘다는 건 모순적)<br>- 실용적 구현 예 없음 | - I/O 요청만 등록하고 스레드는 다른 일 수행<br>- 완료 시 OS가 이벤트/콜백/큐로 알려줌<br>- ex: Linux AIO, io_uring, Windows IOCP |
### 설명 대상의 차이
- {Sync/Async}와 {Blocking/Non-blocking}은 초점을 맞추고 있는 대상이 다릅니다.
- Sync & Async는 프로그램의 일부 control flow에 초점을 맞춥니다. 즉, 리턴 값 또는 결과물을 기다리느냐 그렇지 않느냐입니다.
- Blocking & Non-blocking은 현재의 프로그램을 실행시키는 스레드의 행태에 초점을 맞춥니다. 즉, 스레드가 멈춰 있느냐 그렇지 않느냐입니다.
<br><br/>

### 2 X 2 매트릭스 해석하기
- Sync & Blocking
  - 결과물을 기다릴 때마다 스레드도 멈춰 있습니다.
  - Posix I/O 등의 전통적 파일 리딩, 네트워크 I/O가 이 방식을 택하고 있습니다.
- Sync & Non-blocking
  - 결과물을 기다리지만, 스레드는 멈춰 있지 않습니다. 함수의 호출이 주기적으로 반복됩니다.
  - 결과물 준비 여부와 상관없이 함수가 즉시 리턴되며, 애플리케이션이 루프나 poll로 결과를 반복 확인합니다.
- Async & Blocking
  - 결과물을 기다리지 않는데 스레드는 멈춰 있습니다.
  - 의미가 모순적입니다. Async 한 함수를 호출했으므로 호출된 함수가 바로 리턴을 하지만, 정작 해당 함수를 호출한 스레드는 멈춰 있으므로 뭔가 다른 일을 할 수가 없습니다.
  - 실제 구현 예시를 찾아보기 어려우며, 구현 필요성이 적습니다.
- Async & Non-blocking
  - 결과물을 기다리지 않으며, 스레드 또한 멈춰 있지 않습니다. Node.js, Libuv, Boost Asio IoContext 등의 구현 예시가 존재합니다.
<br><br/>

### select/epoll 함수에 대한 오해 없애기
- select/epoll 함수 자체는 async I/O 가 아닙니다.
- 둘은 응용 프로그램에서 여러 파일 디스크립터(소켓도 파일 디스크립터입니다)의 상태를 감시하는 것에 사용되는 I/O Multiplexing 도구입니다.
- 둘은 필요에 따라서 얼마든지 Sync 함수처럼 또는 Async 함수처럼 사용할 수 있습니다.
- select/epoll 로 이벤트(클라이언트 요청 발생 등)를 감시하고, 실제 I/O(여전히 sync & blocking입니다)는 OS 커널이나 다른 스레드에게 위임하는 것 까지 구현되어야 최종 사용자(클라이언트 또는 서버 개발자) 입장에서는 Async I/O 처럼 보이게 되는 것입니다.
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

### Async - Nonblocking I/O 예시
- 내가 직접 그린 그림이 필요하다.
- 여러 명의 클라이언트들이 들어 왔을 때, 컴퓨터 내부에서 무슨 일이 일어나는지를 설명하는 그림과 단계별 설명이 필요하다.
  - 하나의 메인 스레드와 하나의 워커 스레드를 갖추고 요청이 올 때마다 텍스트 파일을 read 한 후, 그 내용물을 write하여 전송하는 서버가 있습니다.
  - 해당 서버는 2 core CPU 를 갖춘 머신에서 작동합니다.
  - 이러한 상황에서 컴퓨터 내부에서 일어나는 일의 큰 흐름을 정리해 보면 다음과 같습니다.
  - NIC -> DMA -> 소켓 버퍼 -> CPU 인터럽트 -> select/epoll 리턴 -> 응용 프로그램이 소켓을 읽음 -> 요청 해석 -> 워커 스레드에게 파일 리딩 요청 -> 워커 스레드에서 파일 I/O 수행 -> 파일 리딩 완료 이벤트 발생 -> 메인 스레드에서 클라이언트에게 write()
  - 클라이언트 2 명이 거의 동시에 접속하면:
  - 메인 스레드는 select 또는 epoll로 클라이언트 요청을 기다리며 블록 상태입니다.
  -	NIC는 수신 패킷을 소켓 버퍼에 복사 (DMA)하고, CPU에 인터럽트를 발생시킵니다.
  -	커널 인터럽트 핸들러는 메인 메모리 내의 커널 공간에 소켓 버퍼를 준비하고, 메인 스레드의 select(또는 epoll)가 리턴하게 만듭니다.
  -	메인 스레드는 select(또는 epoll) 리턴 후 accept 또는 recv를 호출해 커널 공간 내의 소켓 버퍼 내용을 유저 공간으로 읽고, 요청을 파싱합니다.
  - 요청마다 파일 리딩은 워커 스레드에 맡기며, 메인 스레드는 다시 select(또는 epoll)을 최초 호출하는 부분으로 돌아갑니다. 따라서 다시 새로운 클라이언트를 기다리며 메인 스레드는 블록 상태가 됩니다.
  - 워커 스레드는 OS 시스템 콜(read, pread)을 통해 파일 데이터를 읽으며, 이 과정은 ECF (응용 프로그램 → 커널 전환)로 볼 수 있습니다.
  - 파일 리딩 완료 후, 워커 스레드는 메인 스레드에게 완료를 알림으로서 파일 리딩 완료 이벤트가 발생합니다.
  - 메인 스레드는 이를 감지하고 클라이언트 소켓에 write 시스템 콜을 통해 응답을 보냅니다.
<br><br/><br><br/>



## 왜 용도가 다를까요?
### Sync - Blocking I/O
- 클라이언트 마다 오랫동안 컴퓨팅 리소스를 점유해야 할 때 적합합니다. 결과적으로 클라이언트 1 명당 1 개의 스레드를 할당하게 됩니다.
- connection 별로 state를 유지 및 관리해야 하는 스트리밍 서버의 트래픽이 이러한 양상을 보입니다. RTSP 서버가 대표적입니다.
- 실제 테스트 결과도 존재한다.(내꺼 보여주기)
- [만약 1만 명의 http 클라이언트에게 전부 하나씩 스레드를 할당한다면?]()
  - 과도한 스레드 컨텍스트 스위칭과 I/O 관련 인터럽트들의 발생으로 인해 서버의 기능이 마비될 것입니다.
<br><br/>

### Async - Nonblocking I/O
- 클라이언트 1 명의 컴퓨팅 리소스 점유 시간이 짧고 랜덤할 때 적합합니다. 결과적으로 연결돼 있는 connection(즉, 소켓)은 많지만, 실제로 컴퓨팅 리소스를 장기간 점유하는 connection은 적은 경우에 적합합니다.
- 웹 서버, HTTP 서버의 트래픽이 대표적으로 이러한 양상을 보입니다.
- [만약 1만 명의 고화질 스트리밍 시청자들에게 Async - Nonblocking I/O로 스트리밍 서비스를 제공한다면?]()
  - Async I/O는 connection 수를 늘리고 컨텍스트 스위칭을 최소화하는 장점이 있습니다. 하지만 대용량 데이터를 지속적으로 송신해야 하는 상황에서는 이벤트 루프에서 반복적으로 송신 가능 상태를 감시하고 송신 큐를 관리하는 오버헤드가 발생할 수 있습니다.
  - 따라서 특정 환경에서는 per-client thread 모델이 latency와 throughput 측면에서 더 단순하고 효율적일 수 있습니다.
<br><br/><br><br/>



## 참고자료

- Computer Systems A Programmers's Perspective 3rd edition(2016)
  - Randal E. Bryant, David R. O'Hallaron(
    - 199 ~ 386p : Machine-Level Representation of Programs,
    - 757 ~ 835p : Exceptional Control Flow,
    - 953 ~ 1005p : Network Programming,
    - 1007 ~ 1076p : Concurrent Programming)
<br><br/>
- [C++ RTSP Streaming Server 성능 테스트 결과](https://github.com/DongvinPark/RtspServerInCpp)