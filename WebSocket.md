Before WebSocket 
================== 
Polling or Long Polling(Ajax)

WebSocket
===============
Bi-directional Full Duplex Communication between Web Broswer and Web Server.
Http Environment. RFC 6455

보다 쉽게 상호작용하는 웹 페이지를 만들려면 브라우저와 웹 서버 사이에 더 자유로운 양방향 메시지 송수신(bi-directional full-duplex communication)이 필요하다. 그래서 HTML5 표준안의 일부로 WebSocket API(이후 WebSocket)가 등장했다.

WebSocket은 다른 HTTP 요청과 마찬가지로 80번 포트를 통해 웹 서버에 연결한다.
HTTP 프로토콜의 버전은 1.1이지만 다음 헤더의 예에서 볼 수 있듯이, Upgrade 헤더를 사용하여 웹 서버에 요청한다. 당연한 이야기겠지만 클라이언트인 브라우저와 마찬가지로 웹 서버도 WebSocket 기능을 지원해야한다.

'''bash
GET /... HTTP/1.1
Upgrade: Websocket
Connection: Upgrade 
'''

브라우저는 "Upgrade: WebSocket" 헤더 등과 함께 랜덤하게 생성한 키를 서버에 보낸다. 웹 서버는 이 키를 바탕으로 토큰을 생성한 후 브라우저에 돌려준다. 이런 과정으로 WebSocket 핸드쉐이킹이 이루어진다.

그 뒤 **Protocol Overhead** 방식으로 웹 서버와 브라우저가 데이터를 주고 받는다. **Protocol Overhead** 방식은 여러 TCP 커넥션을 생성하지 않고 하나의 80번 포트 TCP 커넥션을 이용하고, 별도의 헤더 등으로 논리적인 데이터 흐름 단위를 이용하여 여러 개의 커넥션을 맺는 효과를 내는 방식이다.

이런 방식을 사용하기 때문에 방화벽이 있는 환경에서도 무리 없이 WebSocket을 사용할 수 있다.

WebSocket을 지원하는 브라우저와 웹 서버
===================================
클라이언트인 브라우저 중에서는 Chrome, Safari, Firefox, Opera에서 WebSocket을 사용할 수 있으며, 각종 모바일 브라우저에서도 WebSocket을 사용할 수 있다.

웹 서버 중에서는 Apache에서 별도의 모듈을 설치하여 WebSocket을 사용할 수 있다. JEE 환경의 WAS에서는 Jetty, GlassFish에서 WebSocket을 사용할 수 있다. 또한 Node.js에서도 WebSocket을 사용할 수 있다.

출처: https://d2.naver.com/helloworld/1336

Handshake
===============
WebSocket은 HTTP 기반으로 Handshaking을 한다. 어떠한 방식인지 잠깐 훑어보자.

Handshake 요청
```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: v10.stomp, v11.stomp, my-team-custom
Sec-WebSocket-Version: 13
```
Connection: Upgrade : HTTP 사용 방식을 변경하자.
Upgrade : websocket : WebSocket을 사용하자.
Sec-WebSocket-Protocol: xxx, yyy, zzz : WebSocket을 쓰면서 이 중에서 protocol을 골라서 쓰자.
Sec-WebSocket-Key : 보안을 위한 요청 키.

Handshake 응답
===================

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
101 Switching Protocols : Handshake 요청 내용을 기반으로 다음부터 WebSocket으로 통신할 수 있다.
Sec-WebSocket-Accept : 보안을 위한 응답 키 - base64.encode(Sec-WebSocket-Key.concat(GUID))
```

WebSocket Sevrer를 운용할 때의 유의사항
==================================
HTTP에서 동작하나, 그 방식이 HTTP와는 많이 상이하다.
- REST한 방식의 HTTP 통신에서는 많은 URI를 통해 application이 설계된다.
- WebSocket은 하나의 URL을 통해 Connection이 맺어지고, 후에는 해당 Connection으로만 통신한다.

Handshake가 완료되고 Connection을 유지한다.
- 전통적인 HTTP 통신은 요청-응답이 완료되면 Connection을 close한다. 때문에 이론상 하나의 Server가 Port 수의 한계(n<65535)를 넘는 client의 요청을 처리할 수 있다.
- WebSocket은 Connection을 유지하고 있으므로, 가용 Port 수만큼의 Client와 통신할 수 있다.
