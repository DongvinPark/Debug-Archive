# How to Reduce Bitrate Value in 3D VR Streaming
<br/>

## 유의사항
- 본 문서는 구현 과정을 요약한 문서이며, 실제 구현 내용보다 최대한 단순하게 작성했습니다.

<br><br/>
## 개요
- **프로그램 종류** : 고화질(4K) VR 영상 RTSP 스트림 재생용 클라이언트(ExoPlayer를 이용한 안드로이드 앱)
<br><br/>
- **구현 소요 시간** : 2024.11.05 ~ 2025.02.28(116 일)
<br><br/>
- **한 줄 요약** : Hybrid D & S 기술과 LSC 기술을 도입하여 VR 스트리밍의 bitrate를 약 23 Mbps에서 약 7 Mbps로 감소시킴.
<br><br/>
- **AlphaView 3.0** : 비디오 렌더러 2 개를 사용하여 360도 공간 전체를 표현하는 재생 기술. 자세한 정보는 [링크](https://github.com/DongvinPark/Debug-Archive/blob/main/AlphaView3-Impl.md) 참고.
<br><br/>
- **Hybrid D & S** : Hybrid Download and Streaming. 전체 비디오 데이터의 약 절반을 클라이언트가 미리 다운로드 받게 한 후, 나머지 절반만을 스트리밍 함. 클라이언트는 스트리밍 받은 비디오 샘플을 자신의 디바이스에 저장함. 따라서 반복적으로 재생할수록 서버의 스트리밍 bitrate는 낮아짐.
<br><br/>
- **LSC의 뜻** : Looking Sample Control. 사용자가 360도 공간 전체를 전부 인식할 수는 없으므로, 사용자가 보고 있지 않은 각도의 비디오는 스트리밍 서버에서 비디오 샘플을 전송할 필요가 없음. 따라서 서버의 스트리밍 bitrate를 Hybrid D & S만 사용했을 때보다 더 낮출 수 있음.
<br><br/>
- **AVPT 6.1** : AlphaView 3.0 기술의 구현체이자, 고화질 VR 영상 플레이어 어플리케이션. 안드로이드 프로젝트이며, 내부에 ExoPlayer를 사용하고 있음.
<br><br/>
- **cam switching** : AVPT 6.1로 VR 컨텐츠를 재생하는 도중에 시점을 바꾸는 것. 해당 기술을 사용하면 마치 무대 중앙에서 VR 콘서트를 관람하다가 무대의 왼편으로 걸어가서 새로운 시점으로 콘서트를 보는 것 같은 효과를 사용자가 느낄 수 있음.

<br><br/>
## AlphaView 3.0 스트리밍용 데이터 생성 & 전달 아키텍처
<img width="980" alt="Image" src="https://github.com/user-attachments/assets/e6addc08-b480-4bb6-9d51-d0cb8d0a1fa3" /><br>
- 원본 mp4 비디오 파일들이 Transcoder를 통해서 스트리밍용 rtp 데이터 파일들로 변환됨. 이때, FFMPEG가 트랜스코딩에 사용 됨.
- 스트리밍용 파일들을 S3에 업로드 하고, AWS DataSync를 통해서 S3의 파일들이 Network File System의 일종인 EFS(Elastic File System)에 복사 & 동기화 됨.
- 여러 대의 EC2(또는 Elastic Container Service 내의 컨테이너들)에서 돌아가는 RTSP 스트리밍 서버가 EFS에 접근하여 스트리밍용 파일을 읽어들임.
- EFS에서 읽어들인 내용을 바탕으로 RTP 데이터를 클라이언트에게 전달함.

<br><br/>
## Hybrid D & S 의 기본 아이디어
- ***Sequence Diagrame***
    - <img width="841" alt="Image" src="https://github.com/user-attachments/assets/e4ed67f9-3c87-4779-b80d-9b06760e7fb4" />
- ***절반은 미리 다운로드 받게 함***
    - 전체 비디오 데이터 중 약 절반은 클라이언트가 미리 다운로드 받게 함.
    - 나머지 절반은 스트리밍 서버가 전송함.
    - 클라이언트가 받게 되는 데이터의 총량은 Full Streaming과 동일하지만, AWS의 경우 CDN을 통해서 다운로드 받는 것이 EC2 내부의 스트리밍 서버로부터 받는 것보다 약 16.6% 저렴함.
- ***같은 비디오 샘플을 두 번 이상 전송하지 않음***
    - 첫 번째 재생에서 스트리밍 서버가 전송한 비디오 샘플들은 AVPT 6.1에서 재생함과 동시에 클라이언트 디바이스에 저장함.
    - 이렇게 저장된 비디오 샘플은 다음 재생 때 서버에서 다시 전송하지 않기 때문에, 동일 컨텐츠를 같은 클라이언트가 반복적으로 재생할수록 스트리밍 bitrate는 낮아짐.
    - LSC 적용 후 테스트 결과 첫 번째 재생시 약 9 Mbps, 두 번째는 약 6 Mbps, 세 번째는 약 1 Mbps로 bitrate가 낮아졌음.

<br><br/>
## LSC 의 기본 아이디어
- ***안 보고 있는 비디오는 RTP 패킷을 전송하지 않음***
    - 전면 비디오는 240도 공간을 표현하고, 후면 비디오는 120도 공간을 표현함.
    - 사용자의 시선은 거의 대부분 무대와 아티스트가 있는 전면 비디오에 머물러 있음.
    - 따라서, 사용자가 후면 120도 공간을 바라보고 있지 않을 때는 비디오 샘플을 전송하지 않음으로써 불필요한 Data Transfer 를 줄일 수 있음.
- ***LSC 적용 시 발생하는 3 가지 문제점을 해결함***
    - 사용자가 뒤를 보기 시작했을 때 발생하는 후면 비디오 화면 깨짐 문제
    - cam switching 시 음성 끊김 문제
    - cam switching 시 잔존 프레임 문제

<br><br/>
## VR 스트리밍의 한계점
- ***클라우드 서비스의 높은 Network File System(NFS) 비용***
    - 순수 Data Transfer 비용만 놓고 보면 Hybrid D & S와 LSC를 적용한 스트리밍이 Full Download보다 저렴하지만, 스트리밍에 필요한 NFS의 사이즈가 약 3 TB를 넘어가면 Full Download보다 비용이 더 들어가기 시작함.
    - NFS를 사용하지 않는 방법도 있지만, 이 경우 스트리밍을 위한 서버(EC2)와 컨텐츠 컨텐츠 스토리지(Elastic Block Store)를 수동으로 마운트해줘야 하는 불편함이 있음.