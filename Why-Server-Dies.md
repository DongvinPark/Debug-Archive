# 요청이 몰리면 서버가 죽는 이유 - 소켓 뒤에 숨겨진 것들
<br><br/>

## 결론
### 컴퓨터 H/W의 리소스가 고갈됐기 때문입니다.
- 좀 더 구체적으로는, 소켓 뒤에 존재하는 H/W들의 처리 능력이 한계에 도달하여 하나 이상의 H/W에서 병목 현상이 발생했기 때문입니다.
- 이러한 병목현상이 심해지면 운영체제에 의해서 강제로 서버가 종료될 수 있습니다. 대표적으로 Out Of Memory 현상이 있습니다.
<br><br/>

## 서버에 요청이 몰릴 때, 컴퓨터에서 일어나는 일
### 네트워킹과 컴퓨터 H/W
- OSI Layer에서 7 계층에서 호출하는 Socket의 뒤에는 CPU 와 RAM 말고도 Network Interface Card(NIC)라는 하드웨어가 존재합니다.
- NIC에는 NIC buffer가, RAM에는 socket buffer와 user buffer가 존재합니다.
- NIC, DMA, RAM, CPU, Socket, Layer 7 App, nic buffer, socket buffer, user buffer 그림1 필요
<br>

### data flow architecture
- read()
  - 그림 2에 근거한 설명 추가
  - 그림 1에서 리드 방향 화살표 추가한 그림2
- write()
  - 그림 3에 근거한 설명 추가
  - 그림 2에서 라이트 방향 화살표 추가한 그림3
<br>

### 동시접속자 수가 지속적으로 급증한다면? - 수정 필요.
- NIC에서 최대로 유지 가능한 커넥션 개수에 도달하기 시작함
- NIC buffer가 가득 참
- queneing 지연 증가 & loss packet 증가
- DMA를 통해서 커널 내의 socket 버퍼로 패킷을 이동시켜야 하지만 NIC가 보틀넥 되면서 일부 소켓 버퍼가 오랜 시간 동안 채워지지 않기 시작함
- 일부 소켓 버퍼가 채워지지 않기 때문에 그것과 연결된 유저 버퍼도 동일하게 지연 됨
- 결국 해당 버퍼 체인에 의존 중인 클라이언트는 지연 시간이 늘어나고 응답성이 떨어지는 경험을 함
- Asunc read/write일 경우 ECF에 해당하기 때문에 CPU는 스레드 컨텐스트 스위칭 횟수가 급증하고, 그에 따른 오버헤드 또한 급증함. 결국 실제로 일을 처리하는 데 투입할 리소스가 컨텍스트 스위칭 등의 오버헤드보다 작아짐.
- 미디어 스트리밍 서버와 같은 I/O Intensive 앱의 경우 socket buffer, user buffer로 인해서 RAM 내 여유 공간이 급감하고 심할 경우 Out Of Memory 현상으로 운영체제에 의해 서버가 강제 종료될 수 있음
<br>