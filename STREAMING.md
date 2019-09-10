Real Time Messaging Protocol (RTMP)
=============================
- RTMP는 2009년 1월 20일에 발표된 어도비 시스템즈의 독점 통신 프로토콜
- RTMP는 audio, video 및 기타 데이터를 Streaming 할 때 사용. 
  
### RTMP 종류
1. RTMP 
   1. 1935 port 사용, 암호화 되지 않은 프로토콜
   2. 1935 포트로 실패 시, RTMPS(433) or RTMPT(80)으로 재시도.
2. RTMPT(RTMP Tunneled) 
   1. RTMP 데이터를 HTTP로 래핑한다. 당연히 기본 포트 80. 헤더로 인해 데이터 증가.
3. RTMPS(RTMP Secure)
   1. RTMP 데이터를 HTTPS로 매핑 
4. RTMFP(Real Time Media Flow Protocol) 
   1. UDP에서 동작, 항상 암호화된 상태로 전송

### RTP & RTSP
Real Time Streaming Protocol (RTSP)
===================================
- 실시간 스트리밍 프로토콜(RTSP)는 1998년 정의된 통신 규약, RFC2326에 정의.
- **실제 미디어 스트리밍 데이터를 전송하는 프로토콜이 아니며, 사용자가 멀티미디어 스트리밍을 제어할 수 있도록 도와주는 프로토콜**
- 스트리밍 데이터를 제어 하기위해 사용 
- PLAY, PAUSE와 같이 시간 정보를 바탕으로 서버에 접근 - 재생, 일시정지, 빨리감기, 되감기, 재생 위치 변경에 대한 명령 전송
- 대부분 RTSP 서버는 RTP(Realtime Transport Protocol) 규약을 사용, **Trasport Layer**를 통해 실제 Audio/Vido 전송
  

Real Time Protocol (RTP)
===========================
- Transport Layer에 속함 (UDP/IP로 패킷 전송)
- Video, Audio 데이터 전송을 위한 프로토콜
- Header Info : Codec, Order, Timestamp, SSRC Identifier
- 헤더에 코덱 정보 포함.
- Server -> Client 단방향 전송

Real Time Control Protocol (RTCP)
================================
- RTP 데이터의 전송 상태 감시, 세션 관련 정보 전송 - Flow Control
- 양방향 통신