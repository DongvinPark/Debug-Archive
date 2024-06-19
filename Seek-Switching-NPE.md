# Seek Switching Null Pointer Exception Error
<br/>

## 유의사항
- 본 문서는 디버깅 과정을 요약한 문서이며, 실제 구현 내용보다 최대한 단순하게 작성했습니다.

<br><br/>
## 개요
- **프로그램 종류** : 고화질(4K) VR 영상 RTSP 스트림 재생용 클라이언트(ExoPlayer 이용한 안드로이드 앱)
<br><br/>
- **디버깅 소요 시간** : 2024.5.24 ~ 6.5 (13일)
<br><br/>
- **한 줄 요약** : 큐 내 Up Stream 방향 샘플 제거 횟수 결정 로직에서 발생한 오류를 바로 잡음.
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
- 3개 스레드가 비디오 샘플큐 내의 mutex lock을 이용해서 동기화 된 상태로 비디오 샘플 큐에 접근한다.
- 샘플큐 내부에 샘플이 공급, 소비 될 때마다 해당 샘플의 메타데이터가 샘플 메타데이터 배열에 기록된다.
- 샘플 소비 스레드는 리드 기준 인덱스인 K보다 작은 인덱스를 가지는 샘플들을 버린 후(==down stream 샘플 폐기), K 인덱스를 가진 샘플을 소비한다.
- 샘플 공급 스레드는 소켓에서 들어오는 RTP 패킷들을 모아서 하나의 샘플로 만든 후 샘플큐에 샘플들을 추가한다.
- 비디오 스위칭 스레드에 의해서 비디오 스위칭 작업이 진행되면 해당 스레드에서 찾아낸 K 인덱스가 비디오 샘플큐에 전달되고, K 인덱스 값을 기준으로 Up Stream 방향의 샘플들 중 몇 개의 샘플들을 제거할 것인지를 샘플 큐 내부의 메서드에서 계산하여 제거한다.

<br><br/>
## 원인
- **Seek 한 적이 없는 상태에서 스위칭 했을 때**
- 스위칭이 시작되는 샘플의 인덱스와 비디오 샘플 큐 내부에서 사용하는 relative idx가 둘 다 K로 일치하여 적절한 개수의 Up Stream 방향 샘플이 버려진다.
<img width="889" alt="정상적인 스위칭일 때" src="https://github.com/DongvinPark/Debug-Archive/assets/99060708/1c8eff3c-824c-40f4-a7ed-c926a2db2083"><br/>
<br><br/>
- **N번 샘플로 Seek 후 스위칭 했을 때**
- N번 샘플로 Seek를 완료한 후 재생을 시작했을 때, 샘플 큐 내부에서 사용하는 relative idx는 0으로 초기화 된다.
- 이 때, N보다 큰 K의 값을 갖는 샘플에서 스위칭이 일어나면, 샘플 큐 내부에서 사용하는 relative idx인 M보다 훨씬 큰 값인 K가 샘플 큐 내부로 전달 된다.
- M과 K가 서로 불일치 함에 따라 Up Stream 방향의 샘플들을 버리는 횟수가 너무 크거나(정방향 Seek) 너무 적게(역방향 Seek) 설정된다.
- 결과적으로 정방향으로 Seek 했을 경우, 샘플 큐 내부의 샘플들이 전부 버려진다.
- 샘플을 소비해야 하는 코덱 스레드 입장에서는 원하는 샘플을 얻지 못했기 때문에 Null Pointer Exception을 발생시키면서 앱이 작동을 중지한다.
<img width="892" alt="시크 후 스위칭일 때" src="https://github.com/DongvinPark/Debug-Archive/assets/99060708/0a6d54a4-97ff-499f-b026-f443586430f9"><br/>


<br><br/>
## 해결 방법
- **바로 위의 그림에서 보는 바와 같이, Seek가 완료된 시점인 N번 샘플의 인덱스 값을 스위칭이 일어난 시점인 K번 샘플의 인덱스 값에서 빼준 값(== K-N)을 샘플 큐로 전달해주면 된다.**
- 그 결과, 샘플 큐 입장에서는 내부의 relative idx인 M과 (K-N)의 결과 값인 M이 서로 일치하게 되고, Up Stream 방향의 샘플들을 적절한 횟수만큼 제거할 수 있게 된다.