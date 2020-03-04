RSocket Protocol은 어떻게 Reactive Stream을 지원할까?
=======


특정 포맷으로 이루어진 Nelo Log를 특정 단위 stream으로 전송하고 인입 스트림을 실시간 분석하려 했습니다. stream 송신 및 분석 결과 전송 플로우가 reactive하게 작동하는 그림을 그렸고, bi-directional stream 전송 인터페이스로 websocket, grpc등을 고려하던 도중 rsocket protocol을 접하게 되었는데요.

[**RSocket**](https://github.com/rsocket/rsocket)은 Websocket, tcp와 같은 byte stream transport protocol 위에서 동작하는 binary protocol이자 **reactive stream을 지원하는 application protocol**입니다. 


Netflix에서 시작된 rsocket은 msa 환경에서 **오버 헤드가 적은 프로토콜을 통해 http를 대체하기 위해 개발 되었고**, 현재 netifi / facebook / pivotal에 의해 support 되고 있는 오픈 소스입니다. 

[**여기서는**](https://www.netifi.com/rsocket) RSocket에 대해 다음과 같이 설명하고 있는데요.

- **RSocket is a next-generation, reactive, layer 5 application communication protocol for building today's modern cloud-native and microservice applications.**
- **provide a 2x increase in performance over HTTP and gRPC.**  

cpp, js, go, python, kotlin 등을 지원하며, [Java](https://github.com/rsocket/rsocket-java)의 경우 [project reactor](https://projectreactor.io/)를 바탕으로 구현 되었고, [reactor-netty](https://github.com/reactor/reactor-netty)가 transport 역할을 합니다. 

Spring Framework의 경우 5.2 RELEASE에서 RSocket에 대한 [공식 지원](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#rsocket)이 포함 되었습니다. 아직 stable version은 아닙니다만.. 최근 [Spring Blog](https://spring.io/blog/2020/03/02/getting-started-with-rsocket-spring-boot-server)에도 소개된 것을 보니 차세대 신흥 강자? 같은 느낌이 듭니다. 

## Feature, Benefits of the RSocket

먼저 RSocket의 주요 feature입니다.

- **Reactive Stream**
- **Application-Level Flow Control**
- **Session Resumption**

Reactive Stream을 지원하구요. 'Leasing' 그리고 'Reactive Stream'을 통해 **application-level의 flow-control**을 진행합니다. 전자의 경우 조금 생소할 수 있으나 후자의 경우 backpressure를 통해 flow control이 진행 된다는 것을 예상할 수 있습니다. 또한 session resume을 지원합니다. 

개인적으로 이 protocol이 **어떻게 reactive-stream을 지원** 하는가에 대해 궁금증을 가졌고, protocol 명세를 뜯어 보며 알아본 내용을 공유 하고자 합니다 :)

## Interaction Model

RSocket은 OSI Layer 5/6에서 동작하며, 다음과 같은 **4가지 비동기 메시징 interaction model**을 가지고 있습니다.

- **Request - Response** - stream of 1, send one message and receive one back.
- **Request - Stream** - finite stream of many, send one message and receive a stream of messages back.
- **Fire-and-Forget** - no response, send a one-way message.
- **Channel** - bi-directional streams, send streams of messages in both directions.

요 4가지 interaction model을 보니 문득 gRPC와 비슷하다는 생각이 듭니다. 두가지 모두 uni/bi-directional stream을 지원합니다만, http2 위에서 동작하는 gRPC와 조금 더 lower level에서 동작하는 RSocket에는 차이가 있겠죠.

Http/2의 Server push와 비슷한 맥락의 Fire-and-Forget에도 눈길이 가네요!

[**gRPC와 RSocket 비교 분석**](https://medium.com/netifi/differences-between-grpc-and-rsocket-e736c954e60) 또한 매우 흥미롭습니다. (궁금 하신 분들을 위해..!)

&nbsp;

## 용어
어려운 용어는 없고 bi-directional stream을 지원하므로 requester & responder 정의만 숙지하시면 됩니다. 
- **Frame** - request 혹은 response에 포함하는 single message
- **Fragment** - Frame에 포함되기 위해 분할 된 message의 일부 
- **Transport** - RSocket protocol을 전달하는데에 사용하는 transport protocol로, websocket, tcp, aeron 중 하나
- **Stream** - request / response 등 작업 단위
- **Request** - A stream request (위 4가지 interaction model 중 하나).
- **Payload** - A stream message (upstream or downstream). **Reactive Stream 및 Rx 기준 'onNext'에 해당**
- **Complete** - 성공적으로 전송 완료를 알리는 event. **Reactive Stream 및 Rx 기준 'onComplete'에 해당**
- **Client** - connection을 초기화 하는 side
- **Server** - client로 부터의 connection을 accept 하는 side 
- **Connection** - transport session between client and server
- **Requester & Responder** 
    - 초기 connection 생성 이후, client 혹은 server는 위 4가지 interaction 모델을 활용해 interaction을 시작할 수 있음
    - connection 생성 이후에는 client, server 경계가 모호하기에 requester, responder라는 어휘를 활용
    
&nbsp;

## Framing 

RSocket 또한 websocket 처럼 framing을 진행하고, 생성된 RSocket Frame을 WebSocket, TCP 등을 통해 전달합니다. 

framing이 등장하니 벌써 파악하신분도 계실텐데요. 결론적으로 rsocket protocol은 여러 frame과 더불어 frame내의 특정 bit들을 활용해 reactive-stream을 지원합니다. 

### 1. Frame Header Format

프레임의 헤더는 기본적으로 다음과 같은 포맷으로 이루어집니다. (기본 헤더의 **Flags** 부분은 숙지하시면 좋습니다.)

```
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |0|                         Stream ID                           |
    +-----------+-+-+---------------+-------------------------------+
    |Frame Type |I|M|     Flags     |     Depends on Frame Type    ...
    +-------------------------------+
```

- **Stream ID**: (Unsigned 31-bits integer) 현재 프레임의 식별자 (0일 경우 전체 connection을 나타냄) 
- **Frame Type**: (6-bits) 프레임 Type 
- **Flags**: (10-bits) 이 Flag는 프레임 Type에 의존하나 I, M Flag를 위한 공간 제공 
    - (**I**)gnore: Ignore frame it not understood
        - 0: 현재 프레임을 ignore 할 수 없음 / 프레임 판독 불가시 ERROR 프레임을 전송하고 **connection close**
    - (**M**)etadata: metadata 존재 유무 

기본적으로 위와 같은 헤더 포맷을 지니며 각 항목에 대해 조금 더 자세히 알아보겠습니다.

&nbsp;

### 1.1 Stream ID

Stream ID는 requester에 의해 생성되고, 다음과 같은 규약을 지킵니다. 

- Stream ID는 requester 마다 locally unique 해야합니다.
- Stream ID는 31-bits(2^31-1)의 max size를 가집니다. 
- Stream ID 생성은 client는 홀수 ID, 서버는 짝수 ID를 생성해야합니다. 
- requester(client) 측의 Stream ID는 1부터 2씩 증가합니다. (ex. 1, 3, 5, 7, ...)
- responder(server) 측의 Stream ID는 2부터 2씩 증가합니다. (ex. 2, 4, 6, 8 ...) 

또한 Stream ID의 생명주기는 다음과 같습니다.  

- 만약 max size의 Stream ID가 사용 되었다면, Requester는 Stream ID를 초기 값부터 재사용합니다. 
- Responder는 재사용 가능성을 항상 추측하여 설계해야 한다고 합니다.

&nbsp;

### 1.2 Frame Types 

다음은 **가장 중요?** 하다고 볼 수 있는 **RSocket Frame Type**이며 각 Type은 6-bits 크기를 가집니다. 다뤄야 할 프레임이 많고, 어느 정도는 네이밍 자체로 의미를 알 수 있기에 주요 프레임 위주로 살펴 보려고 합니다.

순서는 아래와 같습니다.
- **SETUP Frame**을 시작으로 flow-control에 관련 있는 **LEASE Frame**
- RSocket의 **4가지 interaction model**인 REQUEST_(**RESPONSE/FNF/STREAM/CHANNEL**) 
- PAYLOAD Frame 

네이밍만으로 session resume 관련 프레임(RESUME/RESUME_OK) 또한 확인할 수 있습니다. 추가로 REQUEST_N 프레임의 의미 또한 짐작이 가실텐데요. 아마 **flow-control**에 관여 하는 것 같아 보입니다. 

벌써 어느정도 단서들이 잡히네요.

Type | Value | Description
-----|-------|-------------
RESERVED	| 0x00 |	Reserved
**SETUP**	| 0x01 |	Setup: Sent by client to initiate protocol processing.
**LEASE** |	0x02	| Lease: Sent by Responder to grant the ability to send requests.
KEEPALIVE	| 0x03	| Keepalive: Connection keepalive.
**REQUEST_RESPONSE** |	0x04 |	Request Response: Request single response.
**REQUEST_FNF**	 | 0x05	 | Fire And Forget: A single one-way message.
**REQUEST_STREAM**	 | 0x06	 | Request Stream: Request a completable stream.
**REQUEST_CHANNEL**	 | 0x07	 | Request Channel: Request a completable stream in both directions.
**REQUEST_N** | 	0x08	 | Request N: Request N more items with Reactive Streams semantics.
CANCEL	 | 0x09	 | Cancel Request: Cancel outstanding request.
**PAYLOAD**	 | 0x0A	 | Payload: Payload on a stream. For example, response to a request, or message on a channel.
ERROR	 | 0x0B	 | Error: Error at connection or application level.
METADATA_PUSH	 | 0x0C | 	Metadata: Asynchronous Metadata frame
**RESUME**	 | 0x0D	 | Resume: Replaces SETUP for Resuming Operation (optional)
**RESUME_OK**	 | 0x0E | 	Resume OK : Sent in response to a RESUME if resuming operation possible (optional)
EXT	 | 0x3F | 	Extension Header: Used To Extend more frame types as well as extensions.

&nbsp;

### SETUP Frame (0x01)

먼저 SETUP Frame 입니다. 

이 프레임의 Stream ID 는 항상 0 입니다.
client에 의해 전송되고, 원하는 매개 변수를 server에 전달하는 역할을 합니다. 
네이밍에서 유추 가능하듯이 metadata 형식, data 형식 등(MIME 타입)이 프레임에 포함됩니다. 

```
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                         Stream ID = 0                         |
    +-----------+-+-+-+-+-----------+-------------------------------+
    |Frame Type |0|M|R|L|  Flags    |
    +-----------+-+-+-+-+-----------+-------------------------------+
    |         Major Version         |        Minor Version          |
    +-------------------------------+-------------------------------+
    |0|                 Time Between KEEPALIVE Frames               |
    +---------------------------------------------------------------+
    |0|                       Max Lifetime                          |
    +---------------------------------------------------------------+
    |         Token Length          | Resume Identification Token  ...
    +---------------+-----------------------------------------------+
    |  MIME Length  |   Metadata Encoding MIME Type                ...
    +---------------+-----------------------------------------------+
    |  MIME Length  |     Data Encoding MIME Type                  ...
    +---------------+-----------------------------------------------+
                          Metadata & Setup Payload
```

눈여겨 봐야 할 특징은 다음과 같습니다.

- **Flags**: 글 상단 에서의 헤더 기본 포맷의 (I, M) flags에 추가로 SETUP 프레임에는 **R, L flags**를 위한 공간을 가집니다.
    - (**R**)esume Enable: client의 request는 가능한 경우 재개 / Resume Identifiaction Token이 필요 
    - (**L**)ease: **LEASE 프레임** 수용 여부 
- **Token Length**: (16-bits), Resume Identificaioin Token Length / **(R) flag가 1이면 존재**
- **Resume Identification Token**: client resume identification용 토큰  / **(R) flag가 1이면 존재**
- **Encoding MIME Type**: Data 및 Metadata 인코딩 MIME Type으로 RFC 2045에 지정된 internet media type을 따르는 US-ASCII string
- **Setup Data**: SETUP Frame을 보내는 쪽의 연결 정보를 담은 Payload 

**(R)** flag와 Resume Identification Token을 통해 client와의 connection 재개(**session resumtion**)에 대한 조그마한 단서를 찾았습니다. 또한 **(L)** flag를 통해 flow-control 관련 bit도 확인되네요. LEASE Frame은 바로 뒤에서 설명할 예정입니다.

어쨌든, SETUP 프레임 내에서 rsocket의 주요 feature 중 reactive-stream 보다 **session resumtion & flow-control** 에 대한 단서를 먼저 찾았습니다.

&nbsp;

## LEASE Frame (0x02)

LEASE Frame은 responder측에서 requester에게 전송 되는 프레임이며 다음과 같은 의미를 지닙니다.

- '**일정 시간**' 동안 request를 전송할 수 있음
- '**일정 시간**' 동안 '**얼마 만큼**'의 request 전송할 수 있음 

```
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                         Stream ID = 0                         |
    +-----------+-+-+---------------+-------------------------------+
    |Frame Type |0|M|     Flags     |
    +-----------+-+-+---------------+-------------------------------+
    |0|                       Time-To-Live                          |
    +---------------------------------------------------------------+
    |0|                     Number of Requests                      |
    +---------------------------------------------------------------+
                                Metadata
```

위 정보에 대한 내용은 Time-To-Live / Number of Requests 부분을 확인하면 됩니다. 

- **Time-To-Live** (TTL): (31-bits), LEASE의 유효 시간, 수신 시간부터 시작 (단위: ms) / must be > 0
- **Number of Requests**: (31-bits), 전송 될 수 있는 요청 수 (다음 LEASE가 오기 전까지) / must be > 0

Responder는 requester로 부터의 추가 요청을 중지하고 싶은 경우, 위 value(TTL, NoR) 들이 0인 LEASE Frame을 전송하면 됩니다. 또한 TTL이 expire 되었을 경우, Number of Requests 의 value는 암묵적으로 0 입니다. **REQUEST_N Frame 과 더불어 LEASE Frame 또한 flow-control 관여하는 것으로 보입니다.**

이제 **RSocket의 4가지 interaction model** 관련 프레임을 살펴 볼 차례입니다. 이 4가지 프레임은 connection 생성 후 requester가 responder 에게 **초기 전송하는 Frame** 입니다.

&nbsp; 

## REQUEST_RESPONSE Frame (0x04) & REQUEST_FNF (0x05)

- **REQUEST_RESPONSE**: 4가지 interaction model 중 가장 일반적인 모델로서 '**send one message and receive one back**' 으로 통신합니다.
- **REQUEST_FNF**: response가 없으며 '**send a one-way message**' 입니다. 

두가지 모델 모두 Frame 내부는 간단하고 동일합니다. 

```
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                           Stream ID                           |
    +-----------+-+-+-+-------------+-------------------------------+
    |Frame Type |0|M|F|     Flags   |
    +-------------------------------+
                         Metadata & Request Data
```

- Flags: 
    - (**M**)etadata: metadata 존재 여부 
    - (**F**)ollows: 남아 있는 또 다른 fragment 존재 여부 
    
크게 특별한 것은 없고 딱히 단서를 찾을 수 없었습니다. 

&nbsp;

## REQUEST_STREAM Frame (0x06)

REQUEST_STREAM은 '**send one message and receive a stream of messages back**' 으로 동작하는 interaction model 입니다. 

```
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                           Stream ID                           |
    +-----------+-+-+-+-------------+-------------------------------+
    |Frame Type |0|M|F|    Flags    |
    +-------------------------------+-------------------------------+
    |0|                    Initial Request N                        |
    +---------------------------------------------------------------+
                          Metadata & Request Data
```

- **Flags**: 
    - (**M**)etadata: metadata 존재 여부 
    - (**F**)ollows: 남아 있는 또 다른 fragment 존재 여부 
- **Initial Request N**: (31-bits) 요청할 초기 request 개수를 의미합니다. / must be > 0

Requester는 **REQUEST_STREAM Frame**을 전송하여 초기 Request N을 설정하고, 이후 언제든지 **REQUEST_N Frame**을 전송해 값을 오버로드 할 수 있습니다.

Request - Stream 모델의 경우 다음과 같이 동작할 수 있습니다. 

1. RQ → RS: REQUEST_STREAM (initial request = 2)
2. RS → RQ: PAYLOAD
3. RS → RQ: PAYLOAD
4. RQ → RS: **REQUEST_N** (2)
5. RS → RQ: PAYLOAD
6. RS → RQ: PAYLOAD

바로 다음 Frame 을 보도록 하겠습니다. 

&nbsp;

## REQUEST_CHANNEL Frame (0x07)

REQUEST_CHANNEL은 '**bi-directional streams, send streams of messages in both directions**' 즉 양방향 stream 전송 interaction model 입니다. 

```
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                           Stream ID                           |
    +-----------+-+-+-+-+-----------+-------------------------------+
    |Frame Type |0|M|F|C|  Flags    |
    +-------------------------------+-------------------------------+
    |0|                    Initial Request N                        |
    +---------------------------------------------------------------+
                           Metadata & Request Data
```

눈여겨 봐야 할 부분은 **(C)** flags 입니다. 

- **Flags**: 
    - (**M**)etadata: metadata 존재 여부 
    - (**F**)ollows: 남아 있는 또 다른 fragment 존재 여부 
    - (**C**)omplete: stream completion을 나타냄 
- **Initial Request N**: (31-bits) 요청할 초기 request 개수를 의미합니다. / must be > 0

Requester는 **단 한개의 REQUEST_CHANNEL Frame**을 전송하는데요. 이후 subsequent messages 는 **PAYLOAD Frame**에 실려 전달됩니다. 추가로 Requester가 PAYLOAD Frame을 송신하기 전, Responder는 requester에게 **REQUEST_N Frame**을 항상 보내야합니다. 

Bi-directional stream이므로 responder 또한 REQUEST_N Frame을 requester에게 전송할 수 있고, PAYLOAD Frame을 수신 받을 수 있습니다.  

정상적인 channel의 경우 다음과 같이 동작 할 수 있습니다.

1. RQ -> RS: REQUEST_CHANNEL (initial request = 1) 
2. RS -> RQ: **REQUEST_N**(1)
3. RQ -> RS: **REQUEST_N**(2)
4. RS -> RQ: PAYLOAD
5. RQ -> RS: PAYLOAD
6. RS -> RQ: **COMPLETE**
7. RQ -> RS: PAYLOAD
8. RQ -> RS: **COMPLETE**

Stream 관련 Frame들을 살펴보니 모두 **REQUEST_N Frame**을 통해 flow-control 하고 있는데요. Reactive Stream에서의 **Complete** 또한 볼 수 있습니다. request(n) / complete는 발견 했습니다만.. **next**는 어디에 존재하는 것 일까요?

얼핏 보니 PAYLOAD일 확률이 높아 보입니다. 

&nbsp;

## PAYLOAD Frame (0x0A)

```
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                           Stream ID                           |
    +-----------+-+-+-+-+-+---------+-------------------------------+
    |Frame Type |0|M|F|C|N|  Flags  |
    +-------------------------------+-------------------------------+
                         Metadata & Data
```

눈여겨 봐야 할 부분은 (**C**)omplete, (**N**)ext flags 입니다. 

- **Flags**: 
    - (**M**)etadata: metadata 존재 여부 
    - (**F**)ollows: 남아 있는 또 다른 fragment 존재 여부 
    - (**C**)omplete: stream completion을 나타냄 / **if set, Subscriber의 onComplete() 호출**
    - (**N**)ext: 다음 stream(Payload Data or Metadata) 존재 여부를 나타냄 / **if set, Subscriber의 onNext(Payload) 호출** 

여기서 발견 할 수 있었습니다. **PAYLOAD Frame의 Flags bit를** 이용해 stream complete 및 next signal을 전달하고 있습니다.

## 결론 

RSocket Protocol은 Framing 및 특정 Frame 내의 **Flag bits를 활용해 생각보다 간단하게** Reactive Stream을 지원하고 있습니다. 그리고 Flow-control의 경우 reactive-stream은 REQUEST_N Frame, 그 외의 경우 LEASE Frame을 활용해 flow control을 지원하고 있습니다.

Flow-control 및 Session resumetion 등 다른 feature 관련 단서도 발견 했습니다만 요번에는 자세하게 다루지 못했습니다. 다음번 기회가 된다면 RSocket의 성능적인 내용과 더불어 다른 feature에 대해 다루어 보겠습니다.

성능적인 부분이 궁금 하실 분을 위해 [벤치마킹 링크](https://dzone.com/articles/rsocket-vs-grpc-benchmark)를 남깁니다. 

## Reference
-  https://rsocket.io/docs/Protocol
