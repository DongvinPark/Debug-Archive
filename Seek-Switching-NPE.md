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

<br><br/>
## 원인

<br><br/>
## 해결 방법