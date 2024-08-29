# Used 2 Video Renders in ExoPlayer to Play 360-degree VR Content
<br/>

## 유의사항
- 본 문서는 구현 과정을 요약한 문서이며, 실제 구현 내용보다 최대한 단순하게 작성했습니다.

<br><br/>
## 개요
- **프로그램 종류** : 고화질(4K) VR 영상 RTSP 스트림 재생용 클라이언트(ExoPlayer를 이용한 안드로이드 앱)
<br><br/>
- **구현 소요 시간** : 2024.7.31 ~ 8.23 (24일)
<br><br/>
- **한 줄 요약** : 비디오 렌더러가 1 개였던 기존 AlphaView2.0의 RTSP 파트를 비디오 렌더러 2 개(Front & Rear 비디오 1개씩)를 사용하도록 수정
<br><br/>
- **Switching의 뜻** : 시청자가VR 영상을 보다가 보다가 고개를 돌려서 왼쪽, 오른쪽, 또는 뒤를 봤을 때 클라이언트 앱에서 이에 대응하여 시선의 방향에 부합하는 새로운 화면을 보여주는 것.
<br><br/>
- **AVPT 3.0** : 고화질 VR 영상 플레이어 어플리케이션. 안드로이드 프로젝트이며, 내부에 ExoPlayer를 사용하고 있음.

<br><br/>
## AlphaView 3.0의 특징 & 효과
- ***기존의 AlphaView 2.0 의 한계***
    - [AlphaView 2.0 시연 영상 링크](https://subdued-lentil-b20.notion.site/AlphaView-2-0-Test-763fc8db7bc94808afc0b21f36371815)
    - 비디오 렌더러가 1 개이기 때문에, 사용자가 옆을 볼 경우 'Switching' 과정을 통해서 옆면을 기록한 비디오 파일의 샘플들을 RTSP 서버에게 요청해야 함.
    - 네트워크 속도가 느릴 경우, Switching 시 중간에 검은색 줄이 보이면서 공간이 끊어지는 것처럼 보일 수 있음.
    - 시연 영상 화면의 측면 쪽을 잘 보면, 사용자의 시점이 왼쪽으로 이동할 때 순간 화면에 검은색 줄이 잠깐 깜빡였다가 사라지는 것을 볼 수 있음.
    - 스트리밍에 필요한 비디오 파일의 개수가 많다는 단점이 존재함(12개 필요).
- ***AlphaView 3.0 의 효과***
    - [AlphaView 3.0 시연 영상 링크](https://subdued-lentil-b20.notion.site/AlphaView-3-0-Test-6405d8afc55649deab83b85282dc421d)
    - 비디오 렌더러 2 개를 이용해서 360도 공간을 동일 vsync 내에서 전부 표현하기 때문에, 사용자가 어디를 보든 별도의 네트워킹을 통해서 새로운 비디오 파일의 샘플들을 RTSP 서버에게 요청하는 'Switching'을 해줄 필요가 없음.
    - 따라서 재생 도중 시점을 바꿨을 때 AlphaView 2.0에서 가끔 나타나던 검은줄이 보이지 않음.
    - 스트리밍에 필요한 비디오 파일의 개수가 2 개면 충분함(앞뒤 1개 씩).

<br><br/>
## AlphaView 3.0 스트리밍용 데이터 생성 & 전달 아키텍처
<img width="1146" alt="스크린샷 2024-08-29 오전 10 43 59" src="https://github.com/user-attachments/assets/161124fe-1db0-44c0-aa3b-ea66e9d297e0"><br>
- 원본 mp4 비디오 파일들이 Transcoder를 통해서 스트리밍용 rtp 데이터 파일들로 변환됨. 이때, FFMPEG가 트랜스코딩에 사용 됨.
- 스트리밍용 파일들을 S3에 업로드 하고, AWS DataSync를 통해서 S3의 파일들이 Network File System의 일종인 EFS(Elastic File System)에 복사 & 동기화 됨.
- 여러 대의 EC2(또는 Elastic Container Service 내의 컨테이너들)에서 돌아가는 RTSP 스트리밍 서버가 EFS에 접근하여 스트리밍용 파일을 읽어들임.
- EFS에서 읽어들인 내용을 바탕으로 RTP 데이터를 클라이언트에게 전달함.

<br><br/>
## AVPT 3.0 초기화 & 재생 과정
<img width="1048" alt="스크린샷 2024-08-29 오후 4 09 36" src="https://github.com/user-attachments/assets/2fbd0192-2aae-45b1-97a9-715b4595dcfb"><br>
- AVPT는 RTSP 트랜잭션을 통해서 SDP 메시지를 가져오고, 이에 맞춰서 3 종류의 SampleQueue를 만듦.
- Track Selector는 trackId과 renderer index를 이용해서 적절한 SampleQueue와 renderer를 매핑함.
- AVPT가 RTSP PLAY 요청을 한 후 RTP 패킷들이 들어오면, channel 값에 따라서 적절한 SampleQueue에 RTP 패킷들이 전달되고 SampleQueue에는 샘플들이 누적됨.
    - channel 0, 1, 2 는 각각 front 비디오, audio, rear 비디오 샘플 큐에 대응함.
- ExoPlayer 내의 renderer 들이 샘플들을 SampleQueue로부터 읽어들일 때, Track Selector에 의해서 매핑된 결과에 따른 SampleQueue로부터 샘플들을 읽어들임.
- 각각의 renderer 들이 읽어들인 샘플은 각 renderer에 연결된 코덱에 전달된 후 디코드 되고 최종적으로 화면 또는 스피커로 출력 됨.

<br><br/>
## 개선해야 하는 점.
- ***BitRate 낮추기***
    - 시청자가 항상 뒤를 보고 있거나, 언제가 고개를 좌,우,앞,뒤로 정신없이 움직일 가능성은 낮음.
    - 이러한 상황에서도 현재 AlphaView 3.0에서는 언제나 앞/뒤 비디오 샘플들을 항상 같이 전송함.
    - 이는 bitrate가 불필요하게 높아지는 원인이 되므로 bitrate를 낮추기 위한 방법이 필요함.