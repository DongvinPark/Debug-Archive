# H/W 수준에서 파악해본 Async & Non-blocking I/O



## Sync & Async, Blocking & Non-blocking

### 설명 대상의 차이

- 두 개의 비교축 - 설명하고자 하는 대상이 다르다.
- Sync & Async
- Blocking & Non-blocking
<br><br/>
### 2 X 2 매트릭스 해석하기
- Sync & Blocking
- Sync & Non-blocking
- Async & Blocking
- Async & Nonblocking
<br><br/><br><br/>



## Exceptional Control Flow & Computer H/W

### 역할과 기능
- Bridge 역할
  - 사용자 레벨의 프로그램과 시스템 레벨 프로그램(커널) 간의 연결을 담당한다.
- 다음과 같은 기능을 한다.
  - System call entry/exit
  - Signal delivery + handling
  - Virtual memory (page faults, protection faults)
  - Process/thread context switching
  - Handling of hardware I/O (interrupts)
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
- 클라이언트 마다 오랫동안 컴퓨팅 리소스를 점유해야 할 때
- RTSP 스트리밍 서버다.
실제 테스트 결과도 존재한다.(내꺼 보여주기)
<br><br/>

### Async - Nonblocking I/O
- 클라이언트 1 명의 컴퓨팅 리소스 점유 시간이 짧고 랜덤할 때
- HTTP 서버다. 대표적으로 Node.js가 있다.
<br><br/><br><br/>



## 참고자료

CS : APP 3rd edition
내 테스트 결과