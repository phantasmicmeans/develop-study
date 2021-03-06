### Distributed Streaming Platform Kafka

- Pulish and subscribe to streams of records, similar to message queue or enterprise messaging system.
- Store streams of records in fault-tolerant durable way.
- Process streams of records as they occur.

### Usage
Kafka is generally used for two broad classes of applications

- Building real-time streaming data pipelines that reliably get data between systems or applicatoins
- Building real-time streaming applications that transform or react to the streams of data

### Let's dive in Kafa's capabilities 

First a few concepts:

- Kafka는 하나 이상의 서버에서 클러스터로 실행됨
- Kafka cluster는 topic이라 불리는 category내에 streams of records를 저장함
- 각 record는 key, value, timestamp로 구성됨 

#### Kafka has five core APIs

- **The Producer API**
   - application이 하나 이상의 Kafka topic으로 stream of records를 publish 하도록 함 
- **The Consumer API** 
   - application이 하나 이상의 Kafka topic의 stream of records를 subscribe 하도록 함 
- **The Streams API**
   - application이 stream processor로 act하게 함. 하나 이상의 topic에서 입력 스트림을 consume하고 출력 스트림을 하나 이상의 출력 topic으로 produce하여 입력 스트림을 출력 스트림으로 효과적으로 변환한다.
- **The Connector API**
   - Kafka topic을 기존 애플리케이션이나 데이터 시스템에 연결하는 재사용 가능한 producer나 consumer를 구축하고 실행할 수 있도록 한다. 예를 들어 rdb에 대한 커넥터는 테이블의 모든 변경사항을 캡처할 수 있다.
- **The Admin API**
   - Kafka topic, broker 등에 대한 상태 관리
   
 
Kafka에서의 client-server communication은 TCP protocol을 통해 진행된다. 
 
### Topics and Logs

topic은 category이자 특정 record가 publishing 되어지는 feed name이다. Kafka에서의 topic은 언제나 multi-subscriber이다. 즉, 0 ~ many consumers는 해당 topic에 write 되어진 데이터를 subscribe할 수 있다는 얘기이다. 
 
각 topic에 대해 kafka cluster는 다음과 같은 partitioned log를 유지한다. partition 내의 한 칸을 log라고 생각하면 된다.
각 partition은 ordered, immutable sequence of records로 이루어져 있으며 지속적으로 구조화된 log로 append 된다.

아래 그림은 하나의 topic이 3개의 partition에 분산되어 있는 모습을 나타낸다. 일반적으로 kafka topic은 아래처럼 여러개의 partition에 나눠 write 된다. 

<img src="https://user-images.githubusercontent.com/20153890/83323118-a9c14b00-a297-11ea-85c7-8c8095a6fd9d.png" width=500>

partition내의 records는 각각 offset이라 불리는 sequential id number를 부여받는다. 이 offset은 partitions내에서 각 record를 식별하는 unique한 성질을 가진다.

Kafka cluster는 보존 기간을 두고 publishing 되어진 모든 records를 지속적으로 유지하는데, 해당 기간 내에는 data가 consume 되어도 계속 유지한다. 

예를들어 retention policy 2일로 설정하면 record publish 후 이틀간 사용 가능하며, 이후 공간 확보를 위해 free up space한다.

<img src="https://user-images.githubusercontent.com/20153890/83323391-649e1880-a299-11ea-8734-51275cc3e29c.png" width=430>

log내에서 각 consumer 별 유지되는 유일한 metadata는 해당 consumer에 대한 offset이다.
이 offset은 consumer에 대해 컨트롤 된다. 일반적으로는 consumer가 records를 선형적으로 읽는다.(순차적으로 읽는단 의미)

그러나 각 consumer는 이 offset을 변화시킬 수 있기에, 자신이 원하는 순서에 따라 records를 소비할 수 있다.

예를 들면 각 소비자는 이전에 읽었던 data를 다시 consume 하기 위해 offset을 이전으로 돌려 사용할 수 있고, 가장 최근의 offset으로 설정해 지금 데이터부터 소비할 수 있다.

이 log partitions은 Kafka cluster에 있는 서버를 통해 분산 되고, fault-tolerant를 위해 구성 가능한 여러 서버에 복제된다.

#### Distribution

각 partition은 leader 역할을 하는 하나의 서버와, follower 역할을 하는 0개 이상의 서버를 가진다.
leader는 파티션을 위한 모든 read, write 요청을 처리하는 반면 follower는 leader를 수동적으로 복제한다. 

아래 그림으로 이해하면 쉽다. 

<img src="https://user-images.githubusercontent.com/20153890/83324231-cca32d80-a29e-11ea-96cf-2a61b01fed65.png" width=500>

- 출처: https://engkimbs.tistory.com/691

leader에 문제가 발생하면 follower중 하나가 새로운 leader로 선출된다. 각 서버는 일부 파티션의 leader 역할을 하고 다른 서버에게는 follower 역할을 하는데, 이런식으로 kafka cluster 내 부하 조절을 한다.


### Producers
producer는 특정 topic으로 data를 publish 한다. 또한 어떠한 partition에 어떤 record를 할당할지에 대한 책임을 가진다. 일반적으로 round-robin으로 단순히 부하 균형을 맞추기 위해 수행된다. (다른 partition 함수 사용 가능)

### Consumers

consumer는 consumer group에 포함된다. 특정 topic으로 publish 되어지는 record들은 해당 topic을 subscribe 하는 consumer group 내의 instance들로 전달된다. 각 consumer instance는 각기 다른 process 혹은 다른 machine에 분포되어 있을 수 있다. 

다수의 consumer instance가 같은 consumer group에 포함된 경우, 인입 record는 각 instance들로 load-balancing 된다. 

그러나 각기 다른 consumer group에 존재시, 각 record는 모든 consumer들에게 broadcast 된다. 

<img src="https://user-images.githubusercontent.com/20153890/83324429-37a13400-a2a0-11ea-88bb-35439694f61b.png" width=450>

Kafka cluster는 2대의 서버로 구성되어있고, P0 - P3의 partition을 가진다. 또한 2개의 consumer group으로 구성되어 있다. produce 되어지는 각 record는 round-robin에 의해 각 partition 들에 분배되어 저장 될 것이고, 해당 record에 대한 leader 및 follower 가 결정되어 파티션간 복제를 수행한다.

각 그룹은 모두 여러 consumer instance로 구성되어 있고, 각 consumer instance가 어느 시점에서든 공정하게 data를 consuming 할 수 있도록 한다. 

이는 각 consumer는 **배타적 consumer가** 되어야 하는데, 이 말은 위 그림처럼 각 consumer group내의 instance들이 각기 다른 partition으로 부터 data를 subscribe 해야 한다는 말과 같다.





