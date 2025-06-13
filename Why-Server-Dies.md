# 요청이 몰리면 서버 기능이 마비되는 이유 - 소켓 뒤에 숨겨진 것들
<br><br/>

## 결론
### 역전 현상이 발생하기 때문입니다.
- 좀 더 구체적으로는 클라이언트가 필요로 하는 작업을 처리하는 시간보다, 컨텍스트 스위칭, 인터럽트 처리 등의 작업에 소비하는 시간이 더 길어지기 때문입니다.
<br><br/>

## 서버에 요청이 몰릴 때, 컴퓨터에서 일어나는 일
### 네트워킹과 컴퓨터 H/W
![Image](https://github.com/user-attachments/assets/42eb09c8-400c-41d9-be8c-0b6e1d2c8e0c)
- OSI Layer 7에서 호출하는 Socket의 뒤에는 CPU 와 RAM 말고도 Network Interface Card(NIC)라는 하드웨어가 존재하고, 7 계층의 socket 객체 뒤에는 user buffer 이외에도 두 종류의 buffer가 더 있습니다.
- RAM의 kernel 공간에는 socket buffer가 존재하고, NIC에는 NIC buffer가 존재합니다.
- socket buffer에는 RX 버퍼와 TX 버퍼가 각각 존재하고,
- NIC 내부에도 RX 버퍼와 TX 버퍼가 각각 존재합니다.
- 빨간색 화살표는 socket에 write()를 했을 때 데이터의 흐름이고, 파란색 화살표는 socket에서 read()를 했을 때의 데이터 흐름입니다.
<br>

### 동시접속자 수 급증시 서버에서 발생하는 병목현상의 종류와 원인
- 현대의 네트워크 서버가 이러한 상황에 처하게 되면 아래의 계층 별로 병목 현상이 발생하면서 클라이언트별 응답 대기 시간의 평균과 표준편차가 증가하기 시작합니다.
- NIC 는 수신한 패킷을 DMA를 통해 운영체제 커널의 socket 수신 buffer에 데이터를 직접 복사합니다. 그러나, 다음과 같은 상황에서 병목 현상이 발생할 수 있습니다.
  - socket 수신 buffer가 꽉 차면 NIC는 더 이상 데이터를 복사하지 못합니다. 그 결과 NIC 내의 수신 버퍼가 overflow 되면서 패킷이 버려집니다.
  - 이는 7 계층 앱으로 하여금 재전송 요청을 하게 만들고, 클라이언트 입장에서는 응답 지연을 겪게 됩니다.
  - 이는 대표적인 back pressure 현상으로, 7 계층 앱이 socket buffer 내의 데이터를 충분히 빨리 소비하지 못할 때 발생합니다.
- 7 계층 앱이 read() 또는 write() 등을 호출하면 kernel은 socket buffer의 데이터를 user buffer로 복사합니다. 이때,
  - 7 계층 앱의 비동기 처리 또는 멀티 스레딩이 미흡하거나, CPU 바운드 작업으로 인해 socket buffer 내의 데이터들을 적시에 처리하지 못하면 socket buffer 내의 데이터가 user buffer로 이동하지 못한 채 장시간 대기하게 됩니다.
  - 이러한 상황에서 NIC가 요청을 계속 받으면 해당 요청은 NIC의 buffer overflow로 인해서 버려질 수 있고, 결과적으로 응답시간의 평균과 표준 편차가 증가합니다.
  -  CPU는 동시접속 클라이언트의 수가 급증함에 따라 ECF 처리, 인터럽트 처리, 스레드 컨텐스트 스위칭 횟수가 급증합니다.
     - 이로 인해 서버 컴퓨터 내부에서 한정된 컴퓨팅 리소스를 차지하기 위한 경쟁이 심화되고, 경쟁에서 실패한 thread 또는 ECF 핸들러에 의존하는 클라이언트는 그만큼 더 오랜 시간을 기다려야 합니다.
     - 일정 시점이 지나면 컨텍스트 스위칭에 쓰는 시간이 클라이언트의 요청을 처리하는게 쓰는 시간보다 커지는 역전 현상이 발생하면서 클라이언트가 필요로 하는 서비스 기능이 마비되기 시작합니다.
- 미디어 스트리밍 서버와 같은 I/O Intensive 앱의 경우 socket buffer, user buffer로 인해서 RAM 내 여유 공간이 급감하고 심할 경우 Out Of Memory 현상으로 운영체제에 의해 서버가 강제 종료될 수 있습니다.
<br>

### 참고자료
- Computer Systems A Programmers's Perspective 3rd edition(2016)
  - Randal E. Bryant, David R. O'Hallaron(757 ~ 835p)
- The Linux Programming Interface A Linux and UNIX System Programming Handbook(2010)
  - Michael KerrisK(1149 ~ 1159)
- Linux TCP/IP Stack: Networking For Embedded Systems(2004)
  - Thomas F. Herbert(184 ~ 203)