# REDIS 파이프라이닝

client와 server는 networking link를 통해 연결되어져 있다.
이러한 연결은 매우 빠르거나 매우 느릴 수 있다(두 호스간에 많은 hop이 있는 internet을 거쳐 형성된 커넥션이라면).

client의 packet이 server로 travel하고, server로 부터 다시 client로의 travel time이 있다. 
이러한 시간을 RTT(Round-Trip-Time)라고 한다. 

client가 많은 요청을 in a row(예를들어 같은 리스트에 계속 더하는 요청이라던지).
예를들어 RTT가 250millisecons이면(매우 느린 인터넷 연결), 서버가 초당 100k의  Request를 수용할 지라도 그 프로세스는 1초당 4개의 요청밖에 처리하지 못한다.

인터페이스 used가 loopback(client-server-client) interface이면, RTT는 더 짧다.(예를들어 같은 호스트라면 0.044 milliseconds가 걸린다). 그러나 아직도 perform many writes in a row는 매우 길다.

이를 향상시키기 위한 방법이 있다.
딱 현재와 맞는 상황이다.

## Redis Pipelining
A Request/Response Server는 이전에 client가 요청한 request에 대해 response가 오지 않았더라도, 새로운 request를 보낼 수 있다. 이렇게 우리는 이전 request에 대한 response가 도달하기를 기다리지 않고, multiple command를 전송할 수 있다.  그리고 finally read the replies in a single step

이를 우리는 pipelining이라 부른다. 그리고 이러한 technique는 굉장히 오랫동안 쓰여져 오고있다. 

##  실적
Redis Pipelining 활용해

1. RTT(Rount-Trip-Time)를 줄이자! 
 -> Redis는 TCP 통신을 하는  Server/Client Model이다. 따라서 RTT 시간이 소요된다. 
 각자 1개의 request당 RTT 때문에 많은 request를 보내야 하는 상황에서 굉장히 많은 시간이 소요되었다. 이를 Pipelining으로 간단히 해결할 수 있었다.
  
 -> Network Latency 포함  
### AWS EC2 t2.Large, Seoul Region 기준
1. 7000~ 8000개의 금칙어 리스트를 remote server의 Redis에 저장. 
	- Network Latency포함 
	- 개당 요청시 원격서버 
		- CPU/MEM Usage 0.1 ~ 1% 유지
		- 이전 소요시간 약 30초 ~ 2분 (Network의 상태에 따라 큰 차이를 보임)

	- 파이프라이닝 후 소요시간 : 0.05 ~ 0.07 sec소요.
		결론은 RTT, Network Latency에 따라 확연한 성능저하로 이어질 수 있다. 
		레디스 docu는 10k 정도로 데이터를 잘라 파이프라이닝 하기를 유도한다.
	    이 이상의 데이터는 스트림 전송이 느려질 수 있다.

 
