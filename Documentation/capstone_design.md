# 서론
## 작품선정 배경 및 중요성
### 작품 선정 배경
### 작품의 필요성
## 기존 연구/기술동향 분석
## 개발 목표
## 팀 역할 분담
강태수
- DMR 서버 개발, 안드로이드 애플리케이션 개발

김세현
- DMR 서버 개발, 안드로이드 애플리케이션 개발

구병찬
- 안드로이드 UI 디자인
- 기존 연구/사례 조사

## 개발 일정
조사 및 학습: 12~1월
시스템 설계: 12~2월
구현: 1~4월
테스트 및 데모: 3~5월
문서화 및 발표: 4~6월
최종보고서 작성: 7~9월
## 개발 환경
- Hardware: Galaxy Tab 8.0
- OS: Android 8.0 Oreo
- Frameworks: Android SDK/NDK, GStreamer, Klein

# 본론
## 개발 내용
CCTV 관리 시스템은 직접 영상을 촬영하는 부분인 카메라 모듈, 데이터를 중앙에서 처리 및 녹화하는 녹화기(DMR, Digital Media Recorder), 사용자 애플리케이션으로 나뉜다.
이 중 카메라 모듈은 S1-7 (정수민, et al.) 팀에서 진행하고,
직접 개발하는 부분은 녹화기 겸 서버 역할을 하는 DMR Server와 사용자 애플리케이션인 CCTV Manager App이다.

하드웨어는 두 가지로 나뉘어지는데,
첫번째로 DMR기능을 수행하는 안드로이드 기반의 장치이다.
DMR Server가 직접 실행되는 동시에, 관제실 등에서 사용하기에 적합하도록 CCTV Manager App도 실행하여 사용자가 이 장치를 CCTV 관리에 사용할 수 있도록 한다.
DMR을 안드로이드에서 실행하도록 개발된 경우는 거의 없지만 안드로이드 운영체제의 안정성을 생각하면 안드로이드 기반의 DMR은 기존에 존재하던 시스템보다 훨씬 안정적일 수 밖에 없다.

두번째는 CCTV Manager App을 사용자 기기에 직접 설치하도록 하여 장소에 관계없이 CCTV 관리 시스템에 접근할 수 있도록 한다.

### DMR Server
DMR Server는 크게 두가지의 기능을 수행한다.
첫째로 영상 및 기타 데이터를 저장하는 기능, 즉 DMR(Digital Media Recorder)로서 기능하고,
둘째로 카메라 모듈로부터 받는 실시간 데이터 또는 저장된 데이터를 CCTV Manager App을 통해 사용자에게 전달해주는 기능이다.

DMR Server는 역할에 따라 세가지 부분으로 이루어진다.

#### Web Server
Web Server는 사용자가 직접 접근하는 웹이 아닌,
CCTV Manager App에서 카메라 모듈에 대한 정보를 가져올 때 쓰인다.

여기서 사용하는 웹은 단순한 역할만 하므로 그에 적합한 Klein 라이브러리를 통해 개발하였다.

'/'로 접속할 경우 RTSP Relay Server에 접속할 수 있는 포트정보와 TCP Relay Server에 접속할 수 있는 포트정보, 접속 가능한 카메라 모듈에 대한 정보를 JSON으로 클라이언트에 전달한다.

#### RTSP Relay Server
실시간 영상데이터를 처리할 수 있는 RTSP프로토콜을 이용하여,
카메라로부터 받은 영상데이터를 실시간으로 CCTV Manager App으로 전달함과 동시에 서버 내부 저장소에 저장한다.

구현은 C로 진행하지만, 프로토타입 및 데모에서는 빠른 구현을 위해 python으로 진행한다.

영상 데이터를 다루기 위해 GStreamer 프레임워크를 사용한다.
GStreamer를 사용하면 영상 수신, 처리, 송신을 pipeline을 사용하여 간단하게 처리가 가능하다.

영상 전달 및 처리하는 과정에서 레이턴시를 최대한 없애기 위해 영상데이터 전송은 UDP로 이루어지고, 하드웨어 인코더 및 디코더를 사용하였다.

RTSP Relay Server는 카메라 모듈로부터 영상을 받아오는 RTSP Record Server와 CCTV Manager App에 영상을 전달하는 RTSP Play Server로 이루어진다.

#### TCP Relay Server
영상 데이터뿐만 아닌 카메라 모듈에서 발생하는 Object Detection 데이터를 전달하기 위해서 사용하는 TCP서버이다.
TCP 소켓을 그대로 이용하기 때문에, 직접 프로토콜을 작성하여 카메라 모듈과 CCTV Manager App이 통신할 수 있도록 하였다.


### CCTV Manager App
사용자가 CCTV 관리시스템에 접근할 수 있도록 하는 애플리케이션이다.

## 문제 및 해결방안
문제: RTSP Relay Server에서 영상 송신과 수신을 하나의 서버에서 할 경우 probe buffer event가 발생하지 않는 문제점
  - GStreamer의 RTSP Server모듈은 서버에 접속하는 모드가 두가지로 나뉘어지는데, 클라이언트로부터 영상을 수신하는 모드(GstRtspServer.RTSPTransportMode.RECORD)와 송신하는 모드(GstRtspServer.RTSPTransportMode.PLAY)이다.
    RTSP Relay Server의 주된 동작은 RECORD모드에 해당하는 Pipeline의 출력이 발생하면, 즉 카메라 모듈로부터 영상을 수신하면 발생하는 이벤트에 콜백 함수를 등록하여 그 함수에서 PLAY모드의 Pipeline으로 데이터를 전달하는 것이다.
    그런데 RECORD모드의 클라이언트가 접속한 상태에서 PLAY모드의 클라이언트가 접속하자마자 콜백함수가 실행되지 않는 문제점이 발생하였다. 디버깅을 계속 해본 결과 RECORD모드의 접속 세션과 PLAY모드의 접속 세션끼리 deadlock이 발생하는 문제라는 것을 알았다.
해결방안: RTSP Relay Server에서 RECORD모드 전용 서버와 PLAY모드 전용 서버를 분리하였다.

문제: 안드로이드에서 LibVLC 딜레이 문제 해결힘듦
  - 초기에는 안드로이드에 거의 모든 기능이 지원되는 LibVLC를 사용하여 CCTV Manager App을 구현하기로 하였다. 하지만 LibVLC자체 문제로 영상 수신시의 딜레이를 해결하기 힘들어졌다.
해결방안: GStreamer 사용하여 해결
  - 대신 gstreamer는 안드로이드 지원이 부족하여 직접 라이브러리를 사용하는 부분은 C로 짜야하는 문제점이 있는데
    이것을 영상 처리 및 수신하는 부분은 C, UI는 Kotlin으로 작업하여 웹의 프론트와 백엔드처럼 파트를 나눠 작업함
## 시험 시나리오
## 상세 설계
