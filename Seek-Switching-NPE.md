# Seek Switching Null Pointer Exception Error
<br/>

## 개요
- **프로그램 종류** : 고화질(4K) VR 영상 RTSP 스트림 재생용 클라이언트
<br><br/>
- **디버깅 소요 시간** : 2024.5.24 ~ 6.5 (13일)
<br><br/>
- **한 줄 요약** : 큐 내 포인터 인덱스 값 설정 오류
<br><br/>
- **Seek의 뜻** : 영상의 presentation time을 건너 뛰는 것. 예를 들면, 0분 10초를 보고 있다가 1분 10초로 jump하는 것.
<br><br/>
- **Switching의 뜻** : VR 영상에서 시점을 전환하는 것. 예를 들면, 정면을 보다가 뒤를 보는 것.

<br><br/>
## 증상
- ***Seek 동작 없이 재생하다가 Switching을 했을 때는 정상 동작했는데, Seek 동작 후 Switching을 할 경우, Video SampleQueue에서 Null Pointer Exception이 뜨면서 앱이 종료 됨.***

<br><br/>
## ExoPlayer 내 Video SampleQueue 구조
<img width="901" alt="엑소플레이어샘플큐 구조" src="https://github.com/DongvinPark/Debug-Archive/assets/99060708/e6add7a4-7ad3-441d-80fa-7d5073929f2e"><br/>
- 3개 스레드가 비디오 샘플큐 내의 lock을 이용해서 동기화 된 상태로 비디오 샘플 큐에 접근한다.
- 샘플큐 내부에 샘플이 공급, 소비 될 때마다 해당 샘플의 메타데이터가 샘플 메타데이터 배열에 기록된다.
- 샘플 소비 스레드는 리드 기준 인덱스인 K보다 작은 인덱스를 가지는 샘플들을 버린 후, K 인덱스를 가진 샘플을 소비한다.
- 샘플 공급 스레드는 소켓에서 들어오는 RTP 패킷들을 모아서 하나의 샘플로 만든 후 샘플큐에 샘플들을 추가한다.
- 비디오 스위칭 스레드에 의해서 비디오 스위칭 작업이 진행되면 해당 스레드에서 찾아낸 K 인덱스가 비디오 샘플큐에 전달되고, K 인덱스보다 큰 값을 가지는 Up Stream 방향의 샘플들은 전부 폐기된다.

<br><br/>
## 원인

<br><br/>
## 해결 방법