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
- ***VR 영상의 재생이 일시 정지 된 상태에서 시청자가 고개를 돌려서 Switching을 했을 때, 마치 360도 공간 전체가 멈춘 것처럼 보이게 만드는 것.***
    - [시연 영상 링크](https://subdued-lentil-b20.notion.site/Pause-Switching-Test-e5dff05eba9344b9ba4780313696b2c7)

<br><br/>
## Media Player Architecture
- test contents

<br><br/>
## Step 1 : 스위칭에 걸리는 시간을 단축시키기
- **test contents bold**
- test contents

<br><br/>
## Step 2 : 일시정지 상태에서도 렌더링이 가능하게 만들기
- **test contents bold**
- test contents

<br><br/>
## Step 3 : pause switching 후 다시 재생했을 때 화면이 깨지는 현상 바로잡기
- **test contents bold**
- test contents

<br><br/>
## Step 4 : pause switching 후 재생시킨 다음 일반 switching 했을 때 화면이 멈추는 것 바로잡기
- **test contents bold**
- test contents

<br><br/>
## Step 5 : pause switching을 연이어 실행할 때마다 화면이 조금씩 재생되는 현상 막기
- **test contents bold**
- test contents