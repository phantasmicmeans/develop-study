# Redis
- 오픈소스 in-memory Key, Value 데이터 구조 스토어이다. 
Redis는 **REmote DIctionary Server**의 약자이다. 

- ### 빠른 성능:
	- 데이터를 디스크 또는 SSD에 저장하는 대부분의 DBMS와는 달리 모든 Redis 데이터는 서버의 주 메모리에 저장한다. Redis와 같은 In-memory 데이터베이스는 Disk Access를 할 필요를 없앰으로써 검색 시간으로 인한 지연을 방지하고, CPU 명령을 적게 사용하는 간단한 알고리즘으로 데이터를 액세스 한다. 일반적 작업을 실행하는데 1밀리초 만이 소요된다.

- ### In Memory 데이터 구조
	- Redis를 사용하면 사용자가 다양한 데이터 유형에 매핑되는 키를 저장할 수 있다. 기본적인 데이터 유형은 String으로서, 텍스트 또는 이진 데이터. 최대 크기는  512MB
	#### 지원 데이터 구조 :
	1. Lists of Strings (문자열이 추가된 순서로 유지)
	2. Sets of unordered Strings, 
	3. 점수에 따라 정렬되는 Sorted Sets, 
	3. 필드와 값 목록을 저장하는 Hashes, 
	4. 데이터 세트에서 고유한 항목을 세는 HyperLogLogs를 지원합니다

- ### 복제 및 지속성
	- 마스터 슬레이브 아키텍처를 사용하며 비동기식 복제를 지원한다. 따라서 여러 슬레이브 서버에 복제될 수 있다. 이렇게 주 서버에 장애가 발생하는 경우 요청이 여러 서버로 분산될 수 있으므로 향상된 읽기 성능과 복구 기능 모두 제공.

	- 레디스는 안전성을 제공하기 위해 특정 시점 **SnapShot**(Redis 데이터 세트를 디스크로 복사)과 데이터가 변경될 때 마다 이를 디스크에 저장하는 **Append Only File(AOF)** 생성을 모두 지원한다. 

### 부가 설명
레디스는 기본적으로 Key/Value Store이다. 특정 키 값에 값을 저장하는 구조로 되어있고, 기본적인 PUT/GET Operation을 지원한다.

**단, 이 모든 데이터는 메모리에 저장된다, 이로인해 매우 빠른 write/read속도를 보장한다.** 그리하여 전체 저장 가능한 데이터 용량은 물리적인 메모리 크기를 넘어설 수 없다. 

**데이터 엑세스는 메모리에서 일어나지만 Server restart와 같이 서버가 껏다 켜지는 상황에 데이터 저장을 보장하기 위해 Disk를 영구 저장소로 사용한다**

### Memcached와 다른점은?
- 단순한 Key/Value Store는 이미 memcached가 있다. 그러나 Redis에서는, Value가 단순한 오브젝트가 아닌 **자료구조**를 갖는다! 이때문에 큰 차이를 보인다. 

### 1. String 
	- 일반적인 문자열, 최대 512 M까지 지원. 
	- Text ,Integer, JPEG 같은 Binary File까지 지원.

### 2. Set
	- String 의 집합니다. 여러개의 값을 하나의 Value내에 넣는다. 
	- Set간의 연산을 지원한다!!! - 집합인 만큼 교집합, 합집합, 차이(Differences)를 매우 빠른 시간내에 추출한다.
	
### 3. Sorted Set 
	- Set에 "Score"라는  필드가 추가된 데이터 형으로 Scroe는 일종의 "가중치" 정도로 생각하면 된다.
	- score 오름차순으로 정렬된다. 

### 4. Hashes 
	- hash는 value내에 filed/string value 쌍으로 이루어진 테이블을 저장하는 데이터 구조체이다. RDBMS에서 PK 1개와, String 필드 하나로 이루어진 테이블이라 이해하면 된다.

### 5. List (링크드 리스트 같은 특성)
	- List는 string들의 집합으로 저장되는 데이타 형태는 set과 유사하지만, 일종의 양방향 Linked List라고 생각하면 된다. 
	- List 앞과 뒤에서 PUSH/POP 연산을 이용해서 데이타를 넣거나 뺄 수 있고, 지정된 INDEX 값을 이용하여 지정된 위치에 데이타를 넣거나 뺄 수 있다

![Alt text](./1553579012422.png)




## Persistence 

**Redis는 데이터를 disk에 저장할 수 있다. 서버가 shutdown된 후 restart되어도 disk에 저장해놓은 데이터를 다시 읽어서 메모리에 로드하기 때문에 데이터가 유실되지 않는다.** 

이 데이터를 저장하는 방법이 **Snapshotting** 방식과 **AOF**(Append on File) 두가지가 있다. 

### 1. Snapshotting (RDB) 방식 
- 순간적으로 메모리에 있는 내용은 DISK에 전체를 옮겨 담는 방식,
	##### 1.1 SAVE - Blocking방식으로 순간적으로 Redis의 모든 동작을 정지시키고, 그때의 Snapshot을 Disk에 저장한다. [**동기, 쓰레드 생성 안함, 따라서 작업 시간이 길다.**]

redis.conf
- Save 900 1
- save 300 10
- save 60 10000

위 설정은 900초에 1건이 변경 or 300초에 10건이 변경 or 60초에 10000건이 변경되면 file로 스냅샷 하라는 설정이다. RDB는 스냅샷이지 File에 실시간 저장 되는것이 아니다. 

Save 명령어는 모든 작업을 멈추고 현재 메모리에 대한 데이터를 file에 저장한다. 이 작업은 데이터가 많을수록 긴 시간이 필요하다. save는 동기 방식이기에 새로운 Thread를 생성하지 않는다.

##### 2.2 BGSAVE - Non - Blocking 방식으로 별도의 Process를 띄운 후, 명령어 수행 당시의 메모리 Snapshot을 Disk에 저장한다. 저장 순간에 Redis는 멈추지 않는다. 
	
BGSAVE는 특정 시점에 Redis의 메모리를 데이터 파일로 저장하는 스냅샷이고 비동기 방식으로 Thread를 생성하여 file에 저장한다. 

##### 장점 : 메모리의 snapshot을 그대로 뜬 것이기 때문에, 서버 restart시 snapshot만 load하면 되므로 restart 시간이 빠르다.
##### 단점 : snapshot을 추출하는데 시간이 오래 걸리며, snapshot 추출된후 서버가 down되면 snapshot 추출 이후 데이타는 유실된다. (백업 시점의 데이타만 유지된다는 이야기)


### AOF (Append On File) 방식
- Redis의 모든 Write/update 를 log 파일에 기록하는 방식. 서버가 재 시작될 때 write/update operation을 순차적으로 재 실행해 데이터를 복구한다. 
- operation이 발생할 때 마다 매번 기록하기 때문에 항상 현재 시점까지 기록할 수 있다.
- 기본적으로 Non-blocking Call이다. 
- Non-Blocking 비동기로 동작하기에 AOF를 위한 **다른 쓰레드들이 존재한다!!!**
(AOF를 위한 Thread가 존재한다. 그러나 **명령을 수행하기 위한 Thread는 1개이기에 Redis를 Single Thread라 하는것임.**)

	##### 장점 : Log file에 대해서 append만 하기 때문에, log write 속도가 빠르며, 어느 시점에 server가 down되더라도 데이타 유실이 발생하지 않는다.
	##### 단점 : 모든 write/update operation에 대해서 log를 남기기 때문에 로그 데이타 양이 RDB 방식에 비해서 과대하게 크며, 복구시 저장된 write/update operation을 다시 replay 하기 때문에 restart속도가 느리다.

### 권장 사항
- 두가지 방식을 혼용하여 사용하는 것이 바람직하다. 주기적으로 Snapshot으로 백업하고, 다음 snapshot까지의 저장을 AOF로 수행하는것이 좋다. 이렇게 하면 서버가 restart될 때, Snapshot을 reload하고, 소량의 AOF만 replay하면 된다. 
- restart 시간을 절약하고 데이터의 유실을 방지할 수 있다.
- 

#  Replication Topology
1. Master - Slave구조. 
- Master /Slave Replication이란, Redis의 Master node에 Write된 내용을 복제를 통해 slave Node에 복제하는 것을 정의한다. 
- Master Node는 N개의 Slave Node를 가질수 있으며, 그 Slave Node는 또 다른 Slave Node를 가질 수 있다.
- **Master에서는 Read/Write가 가능하고, Slave에서는 Read Only이다.**

##### 이 master/slave 간의 복제는 Non-blocking 상태로 이루어진다. 즉 master node에서 write나 query 연산을 하고 있을 때도 background로 slave node에 데이타를 복사하고 있다는 이야기고, 이는 master/slave node간의 데이타 불일치성을 유발할 수 있다는 이야기이기도 하다.
##### master node에 write한 데이타가 slave node에 복제중이라면 slave node에서 데이타를 조회할 경우 이전의 데이타가 조회될 수 있다.

# Query Off Loading을 통한 성능 향상
- 그러면 이 master/slave replication을 통해서 무엇을 할 수 있냐? 성능을 높일 수 있다. 동시접속자수나 처리 속도를 늘릴 수 있다. (데이타 저장 용량은 늘릴 수 없다.) 이를 위해서 Query Off Loading이라는 기법을 사용하는데
- **Query Off Loading**은 master node는 write only, slave node는 read only 로 사용하는 방법이다.

- 단지 redis에서만 사용하는 기법이 아니라, Oracle,MySQL과 같은 RDBMS에서도 많이 사용하는 아키텍쳐 패턴이다.
대부분의 DB 트렌젝션은 웹시스템의 경우 write가 10~20%, read가 70~90% 선이기 때문에, read 트렌젝션을 분산 시킨다면, 처리 시간과 속도를 비약적으로 증가 시킬 수 있다. 특히 redis의 경우 value에 대한 여러가지 연산(합집합,교집합,Range Query)등을 수행하기 때문에, 단순 PUT/GET만 하는 NoSQL이나 memcached에 비해서 read에 사용되는 resource의 양이 상대적으로 높기 때문에 redis의 성능을 높이기 위해서 효과적인 방법이다.


# Sharding 을 통한 용량 확장
- redis가 클러스터링을 통한 확장성을 제공하지 않는다면, 데이타의 용량이 늘어나면 어떤 방법으로 redis를 확장해야 할까?

- 일반적으로 **Sharding**이라는 아키텍쳐를 이용한다. Sharding은 Query Off loading과 마친가지로, redis 뿐만 아니라 일반적인 RDBMS나 다른 NoSQL에서도 많이 사용하는 아키텍쳐로 내용 자체는 간단하다.

- 여러개의 redis 서버를 구성한 후에, 데이타를 일정 구역별로 나눠서 저장하는 것이다. 예를 들어 숫자를 key로 하는 데이타가 있을때 아래와 그림과 같이 redis 서버별로 저장하는 key 대역폭을 정해놓은 후에, 나눠서 저장한다.

**데이타 분산에 대한 통제권은 client가 가지며 client에서 애플리케이션 로직으로 처리한다.**



# Expiration
**redis는 데이타에 대해서 생명주기를 정해서 일정 시간이 지나면 자동으로 삭제되게 할 수 있다.**
- redis가 expire된 데이타를 삭제 하는 정책은 내부적으로 Active와 Passive 두 가지 방법을 사용한다.
	1. **Active** 방식은 Client가 expired된 데이타에 접근하려고 했을 때, 그때 체크해서 지우는 방법이 있고
	2. **Passive** 방식은 주기적으로 key들을 random으로 100개만 (전부가 아니라) 스캔해서 지우는 방식이 이다.

	- expired time이 지난 후 클라이언트에 의해서 접근 되지 않은 데이타는 Active 방식으로 인해서 지워지지 않고 Passive 방식으로 지워져야 하는데, 이 경우 Passive 방식의 경우 전체 데이타를 scan하는 것이 아니기 때문에, redis에는 항상 expired 되었으나 지워지지 않는 garbage 데이타가 존재할 수 있는 원인이 된다.


# Cluster (클러스터)
	- HA, 샤딩 지원
	- 데이터 셋을 자동으로 여러 노드들에 나누어 저장해준다.
	- Redis Cluster 기능을 지원하는 Client를 사용해야 데이터 액세스 시에 올바른 노드로 redirect가 가능하다.
	- 노드들의 동작방식
		- Server Client : 7001 포트,
		- Cluster Bus : Server Client Port + 10000 : => 10701 포트 
			- 자체적인 바이너리 프로토콜을 통해 node-to-node 통신을 한다. 
			- Failure detection. Configuration Update, Fail Over Authorization 등을 수행한다.
			- 각 노드들은 클러스터에 속한 다른 노드들에 대한 정보를 모두 갖고 있다.

### 샤딩 방식
	 - Hash Slot이라는 개념을 도입해서 사용.
	 <HashSlot> :
	 - 결정방법 CRC16(key) mod 16384
	 - 노드별로 자유롭게 Hash Slot을 할당 가능 
			 - EX) 
				 - Node A Contains hash slot 0 to 5500.
				 - Node B Contains hash slot 5501 to 11000.
				 - Node C Contains hash slot 11001 to 16383.
		- 운영 중단 없이 Hash Slot을 다른 노드로 이동시키는 것이 가능.
			- add/remove nodes
			- 노드별 HashSlot 할당량 조정가능
			
**Multiple Key Operation을 수행하려면 모든 키 값이 같은 Hash Slot에 들어있어야함**
-> 이를 보장하기 위해 **hashtag**라는 개념 도입. 
	- {} 안에 있는 값으로만 Hash 계산
	- {foo}_my_key
	- {foo}_your_key

### Replication & FailOver 
	FailOver를 위해 클러스터의 각 노드를 N대로 구성가능
	Master 1대 / Slave(N-1)대
	
-> 클라이언트는 클러스터에 어떻게 연결해서 데이터를 조회하나?
	- 1. Redis Client는 클러스터 내의 어떤 노드에 쿼리를 날려도 됨.(슬레이브에 날려도 됨)
	- 2. EX) GET my_key
	- 3. 쿼리를 받은 노드가 해당 쿼리를 분석 -> 해당 키를 자신이 갖고 있다면 바로 찾아서 값을 리턴한다 / 그렇지 않은 경우 키를 저장하고 있는 노드의 정보를 리턴.
	- 해당 키를 가지고 있지 않은경우는 클라이언트가 Response를 토대로 다시 쿼리를 보내야함. ex) MOVED 3999 127.0.01:7001 

