# Pause Switching Implementation
<br/>

## 유의사항
- 본 문서는 개발 & 디버깅 과정을 요약한 문서이며, 실제 구현 내용보다 최대한 단순하게 작성했습니다.

<br><br/>
## 개요
- **프로그램 종류** : 고화질(4K) VR 영상 RTSP 스트림 재생용 클라이언트(ExoPlayer 이용한 안드로이드 앱)
<br><br/>
- **Streaming Protocol** : RTSP
<br><br/>
- **개발 & 디기버깅 소요 시간** : 2024.6.15 ~ 7.17 (33일)
<br><br/>
- **일반 Switching의 뜻** : VR 영상을 재생하는 도중에 시점을 전환하는 것. 예를 들면, 정면 영상을 보다가 뒷면 영상을 보는 것.
<br><br/>
- **Pause Switching의 뜻** : VR 영상을 일시정지 시킨 상태에서 시점을 전환하는 것. 이 상태에서 고개를 좌우로 돌려보면 눈에 들어오는 360도 공간 전체가 일시정지 된 것처럼 보임.

<br><br/>
## 개발한 기능
- ***VR 영상의 재생이 일시 정지 된 상태에서 시청자가 고개를 돌려서 Pause Switching을 했을 때, 마치 360도 공간 전체가 멈춘 것처럼 보이게 만드는 것.***
    - [시연 영상 링크](https://subdued-lentil-b20.notion.site/Pause-Switching-Test-e5dff05eba9344b9ba4780313696b2c7)

<br><br/>
## ExoPlayer RTSP Part Architecture
<img width="1013" alt="스크린샷 2024-07-18 오전 9 51 24" src="https://github.com/user-attachments/assets/2bb1c7d2-0d4a-4eeb-bca9-0fd81f774bf1"><br>
- **Source** : RTSP 스트림을 제공하는 미디어 서버를 뜻한다.
- **Load Control** : RTSP 프로토콜로 미디어 서버와 통신하면서 스트림의 재생, 정지, 일시정지, seek, switching 동작을 수행한다. 필요에 따라서 Loader의 작동을 중지시키거나, Sample Queue를 reset 시킨 후 전부 비운다.
- **Loader** : RTSP 서버로부터 받은 RTP 패킷을 샘플 단위로 모아서 sample queue에 커밋한다.
- **Sample Queue & Track Selector** : 샘플들을 timestamp 순서에 맞게 저장하고 있다가 Renderer에게 넘겨 준다.
- **Renderer** : sample queue에서 읽어들인 샘플 데이터를 코덱에게 넘겨준다.
- **Codec** : renderer에게 넘겨 받은 샘플을 디코드하여 frame으로 만든 후, renderer에게 디코드 완료를 알린다.
- **Surface** : 코덱을 거쳐서 생성된 프레임 데이터를 저장한다. 그래픽 버퍼이며 최종 디스플레이에 사용된다.

<br><br/>
## Step 1 : Pause Switching에 걸리는 시간을 단축시키기
<img width="1029" alt="스크린샷 2024-07-18 오후 1 31 32" src="https://github.com/user-attachments/assets/895eeb08-f7e8-43a9-8d0e-d092a2066e19"><br>
<img width="1048" alt="스크린샷 2024-07-18 오후 2 55 09" src="https://github.com/user-attachments/assets/81cedc67-f0c1-4394-b4a0-f9b937601889"><br>
<img width="1046" alt="스크린샷 2024-07-18 오후 3 03 30" src="https://github.com/user-attachments/assets/646346a1-7344-4d78-a863-1280afecf840"><br>
- **Pause Switching에서 추가적으로 필요하는 Seek Play를 Source 서버와의 트랜잭션 없이 시작하게 만듦.**
- 두 번의 네트워크 트랜잭션(Switching, Seep Play)이 한 번으로 줄어들어서 pause switching의 전체 수행시간이 단축됨.
- seek play가 시작됨으로써, pause 상태가 강제로 해제되고 renderer가 작동하기 시작함.
- pause switching 완료 후, seek play를 다시 중지시킴.

<br><br/>
## Step 2 : pause switching 후 다시 재생했을 때 화면이 깨지는 현상 바로잡기
<img width="1027" alt="스크린샷 2024-07-18 오후 1 34 11" src="https://github.com/user-attachments/assets/e34cff83-7c0e-472b-93ef-75d48331b3cc"><br>
<img width="950" alt="스크린샷 2024-07-18 오후 3 37 08" src="https://github.com/user-attachments/assets/3f954759-7f25-44a6-89c5-57d6d36dc4fb"><br>
- **Pause Switching 때 전송하는 샘플의 개수를 제한시킴.**
- pause switching 때 전송하는 샘플의 개수를 제한하지 않고 서버에서 전송을 계속하게 만들면, pause switching이 종료된 후 유저가 playback 시켰을 때 전송되는 샘플들과 뒤섞이면서 화면이 깨지는 현상이 발생함.

<br><br/>
## Step 3 : pause switching 후 재생시킨 다음 일반 switching 했을 때 화면이 멈추는 것 바로잡기
<img width="1019" alt="스크린샷 2024-07-18 오후 1 35 28" src="https://github.com/user-attachments/assets/4cf8c228-8160-4182-8a6b-5620a0b84631"><br>
- **코덱에서 마지막으로 디코드한 샘플의 인덱스를 기준으로, 바로 다음 인덱스의 샘플부터 Source 서버에서 전송하게 만듦.**
- 예들 들어서, pause switching 후 다시 playback해서 재생 중인 상태에서 코덱에서 11번 샘플을 디코드 완료했다고 가정.
- 바로 이 타이밍에 일반 switching이 시작되었다면, Source 서버로 하여금 12번 샘플부터 전송하게 만듦.

<br><br/>
## Step 4 : pause switching을 연이어 실행할 때마다 화면이 조금씩 재생되는 현상 막기
<img width="1019" alt="스크린샷 2024-07-18 오후 1 35 28" src="https://github.com/user-attachments/assets/4cf8c228-8160-4182-8a6b-5620a0b84631"><br>
<img width="1001" alt="스크린샷 2024-07-18 오후 4 22 22" src="https://github.com/user-attachments/assets/1f074ebf-4464-4e34-8618-57c502d5681b"><br>
- **최초의 pause switcing 때 Source 서버에서 전송한 첫 샘플의 인덱스를 다음 pause switching 때도 재활용함.**
- 이러한 재활용 조치가 없을 경우, pause switching이 연속적으로 실행되는 상황에서 8 프레임씩 영상이 정방향 재생되는 현상이 발생함. 이 상태로 pause switching만 계속 반복하면 영상이 결국 끝까지 재생 됨.