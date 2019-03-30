
Docker Container Network에 대한 이해
==================================

*by S.M.LEE*

## <docker0, container network의 구조> ##
 
**NOTE**

* 아래 그림은 docker의 network 구조를 간단히 도식화한 것이다.

* 여기서는 docker를 설치하면 가장 먼저 볼 수 있는 docker0 interface와 container network에 대해 알아본다.

 ![image](https://user-images.githubusercontent.com/20153890/40031808-49f6a9f2-582c-11e8-9c51-052ad4ddcbcf.png)

## 1. docker0 interface ##
**Docker host를 설치한 후 host의 network interface를 보면, docker 0 이라는 interface를 볼 수 있다.**


>	-     $ifconfig 


![image](https://user-images.githubusercontent.com/20153890/40032017-392a7c56-582d-11e8-956c-5dea8308f525.png)

이 docker0 interface의 특징은, 
> -   IP는 자동으로 172.17.0.1로 배정된다. 
> -   IP는 DHCP로 자동할당이 되는 것이 아니고, docker 내부 로직에 따라 자동할당 된다.
> -   이 docker0은 일반적은 interface가 아니고, virtual ethernet bridge이다.

즉 docker0은 Container가 통신하기 위한 virtual bridge라고 볼 수 있다.

하나의 Container가 생성시, 이 bridge에 container의 interface가 하나씩 binding되는 형태이다.
그리고 container가 running될 때 마다 vethXXXX라는 이름의 interface가 attach되는 형태이다.

결론적으로 container가 외부로 통신할 때는 무조건 docker0 interface를 지나야 한다.

또한 docker0의 ip는 자동으로 172.17.0.1로 설정되고, subnet mask는 255.255.0.0 (172.17.0.0/16) 으로 설정된다.
이 subnet 정보는 container가 생성될 때마다 container가 할당받게 될 IP의 range를 결정하게 된다.

**즉 모든 container는 172.17.XX.YY 대역에서 IP를 하나씩 할당받게 된다.**

 

## 2. container Network의 구조 ##

docker는 host에 container를 생성하게 되었을때, 각 conatiner는 격리된 공간을 할당받는다.

그렇다면 이 격리된 container는 어떻게 외부(또 다른 container로) 와 통신을 할까?
  
![image](https://user-images.githubusercontent.com/20153890/40032150-c9f5701a-582d-11e8-8813-1c292cba2e71.png)


먼저 container가 생성되면, 해당 container에는 pair (peer) interface라고 하는 한 쌍의 interface가 생성된다.
이 pair interface는 두 interface가 한쌍으로 구성되고, 마치 direct로 연결한 두 대의 PC처럼 packet을 주고받는다.

즉 container 생성시, pair interface의 한쪽은 container내부에 eh0이라는 이름으로 할당되고,
다른 한쪽은 vethXXX라는 이름으로 docker0 bridge에 binding된다.

> -     $ip link
 
![image](https://user-images.githubusercontent.com/20153890/40032391-e8fee030-582e-11e8-8a21-2a4290d5694e.png)

container를 하나 올린 상태에서 link를 확인해보면, running중인 container는 vethXXX라는 이름으로 docker0 bridge에 연결되어 있는 것을 확인 할 수 있다.
 
 
그럼 container내부에 할당된 eth0 interface는 어떻게 확인할까?
이는 해당 namespace에서만 보이도록 격리되어 있으므로 running중인 container 내부에서 확인해야 한다.

> -	    $docker exec {container id} ifconfig eth0

![image](https://user-images.githubusercontent.com/20153890/40032438-1c1dccba-582f-11e8-8976-2823db34157e.png)

container안에 외부 통신을 위한 eth0 interface가 있는 것을 볼 수 있고, 
172.17.0.2라는 IP가 할당되었으며, netmask도 255.255.0.0으로 설정되어져 있는 것을 볼 수 있다.

**그리고 이 container의 gateway는 docker0에 설정된 ip인 172.17.0.1로 되어져 있다.**

> -	    $docker exec {container id} route
 
![image](https://user-images.githubusercontent.com/20153890/40033675-6247c2ea-5834-11e8-95f2-98df9023822e.png)

Container 내부의 모든 packet은 default인 172.17.0.1(docker0의 ip)로 전송된다.



지금까지 docker0 bridge mode, container network에 대해 설명을 해보았다.

brige모드는 docker network의 default설정이자, 가장 많이 쓰이는 방식이다.


## <container network 외부 통신 구조 > ##


여기서는 container가 외부와 연결되기 위해 어떤 구조로 동작하는지 알아보겠다.

## 1. container port 를 외부로 노출시키기 ##

container 생성시, 각 container에는 격리된 네트워크 환경이 부여된다. 그리고 각 container는 Docker host와 통신을 위해
linux bridge방식으로 binding되어져 있는 형태이다. 따라서 Docker host내의 container들은 자신이 할당받은 private ip(172.0.~)을 통해 
통신이 가능하다.

예를들어 Maven을 이용해 Spring boot project를 build하고, 생성된 jar파일을 실행하여 WAS를 container로 서비스한다고 하자.
이 WAS의 port가 8761이라고 할때 이 port는 반드시 외부와 통신이 되어야만 서비스가 가능하다.

그러나 container는 private ip를 가지고 있고, bridge형태로 Docker host와 연결되어 있기 때문에 직접적으로 외부와의 통신은 불가능하다.

따라서 외부와의 통신을 위해 container를 외부로 노출할 port를 지정해주어야 한다.

> -	    $docker run -d -p 8761:8761 {container id or Image} 

-p option은 container port와 외부 port를 binding할때 사용하는 option이다.

> -	    $docker ps | awk  '/PORTS/{print $7} {print$11}'

![image](https://user-images.githubusercontent.com/20153890/40152472-5234faac-59c0-11e8-9ae5-1a1ca90bac99.png)


위 명령어를 실행하면, 외부 port와 binding 되어 있는것을 볼 수 있다.
이는 Docker host의 8761 port로 요청이 들어오면 실행중인 container의 8761 port로 forwarding하겠다는 의미이다.


> -	    $netstat -tnlp | grep 8761


![image](https://user-images.githubusercontent.com/20153890/40152624-fbb70048-59c0-11e8-959f-08f19ade5395.png)


그리고 다음과 같이 LISTEN 중인 상태를 확인하면, 8761 port가 docker-proxy라는 process에 의해 활성화 되어져 있는것을 볼 수 있다.


## 2. docker-proxy란 ##

Docker host로 들어온 요청을 container로 넘긴다. 즉 host가 받은 패킷을 그대로 container의 port로 넘기는 역할을 한다.

container 실행시, port를 외부에 노출시키면 docker host에는 docker-proxy라는 process가 생성된다.
예를들어 두개의 port를 외부에 노출시키면 , 두개의 docker-proxy process가 생성되는 것이다. 

하지만 docker-proxy와 관계없이 docker host의 iptables에 의해 container로 패킷이 전달된다.
따라서 docker-prxoy process를 kill해도 패킷이 container로 전달되는데 문제가 없다.

## 3. docker iptables ##

> -	    $iptables -t nat -L -n


![image](https://user-images.githubusercontent.com/20153890/40152882-1dc7376a-59c2-11e8-8c52-be778ecb04f0.png)

Docker host의 iptables 내역이다.

들어온 요청은 PREROUTING -> DOCKER Chain으로 전달된다. DOCKER Chain부분을 살펴보면 tcp dpt:8761 to:172.17.0.2:8761 라인을 볼수 있다.
이는 DNAT로 8761 port로 들어온 패킷 -> 172.17.0.2인 IP를 가진 container의 8761 port로 port forwading 된다는 의미이다.
