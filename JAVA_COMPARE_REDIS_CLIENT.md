Jedis 보다 Lettuce 를 쓰자
============================

Java의 Redis Client는 크게 2가지가 있습니다.

1. Jedis
2. Lettuce

둘 모두 몇천개의 Star를 가질만큼 유명한 오픈소스입니다. 

### 0.프로젝트 환경
의존성 환경은 아래와 같습니다.

- Spring Boot 2.1.4
- Spring Boot Data Redis 2.1.4
- Jedis 2.9.0
- Lettuce 5.1.6

### 1. EC2 사양
Spring Boot가 실행되고 Redis로 요청할 EC2의 사양은 아래와 같습니다.

- r5d.4xLarge X 4대


### 2. Redis 사양

- Redis 5.0.3

### 3. Test
그리고 테스트에 사용될 Redis Entity 코드는 아래와 같습니다.

```java
@ToString
@Getter
@RedisHash("availablePoint")
public class AvailablePoint implements Serializable {

    @Id
    private String id; // userId
    private Long point;
    private LocalDateTime refreshTime;

    @Builder
    public AvailablePoint(String id, Long point, LocalDateTime refreshTime) {
        this.id = id;
        this.point = point;
        this.refreshTime = refreshTime;
    }
}
public interface AvailablePointRedisRepository extends CrudRepository<AvailablePoint, String> {
}
```

임의의 데이터를 저장하고, 가져올 Controller 코드는 아래와 같습니다.

```java
@Slf4j
@RequiredArgsConstructor
@RestController
public class ApiController {
    private final AvailablePointRedisRepository availablePointRedisRepository;

    @GetMapping("/")
    public String ok () {
        return "ok";
    }

    @GetMapping("/save")
    public String save(){
        String randomId = createId();
        LocalDateTime now = LocalDateTime.now();

        AvailablePoint availablePoint = AvailablePoint.builder()
                .id(randomId)
                .point(1L)
                .refreshTime(now)
                .build();

        log.info(">>>>>>> [save] availablePoint={}", availablePoint);

        availablePointRedisRepository.save(availablePoint);

        return "save";
    }


    @GetMapping("/get")
    public long get () {
        String id = createId();
        return availablePointRedisRepository.findById(id)
                .map(AvailablePoint::getPoint)
                .orElse(0L);
    }

    // 임의의 키를 생성하기 위해 1 ~ 1_000_000_000 사이 랜덤값 생성
    private String createId() {
        SplittableRandom random = new SplittableRandom();
        return String.valueOf(random.nextInt(1, 1_000_000_000));
    }
}
```

위 /save API를 통해 테스트할 데이터를 Redis에 적재해서 사용합니다. 
성능 테스트는 /get API를 통해 진행합니다.


VUser 740명으로 시도합니다.

agent는 5대
각 agent 별 148명을 지정했습니다.
부하 테스트 시간은 3분

테스트용 데이터는 대략 1천만건을 적재하였습니다.

### Jedis 

##### 성능 테스트 결과

- 부하를 주는 Ngrinder의 Agent는 평균 CPU가 15 ~ 17%를 유지했습니다.
- TPS는 3만이 나왔습니다.
- 핀포인트로 측정한 응답속도 는 100ms가 나왔습니다
- 그리고 Redis의 Connection 개수는 35개를 유지했습니다.
- 마지막으로 Redis의 CPU는 20%가 최고치였습니다.

생각보다 성능이 잘나오지 않은것 같죠?

부하를 주는 Ngrinder의 Agent CPU가 20%를 넘지 못했습니다. 
즉, Redis 응답속도가 좀 더 높다면 훨씬 더 높은 TPS가 나올수 있다는 이야기입니다.

자 그럼 같은 사양으로 Connection Pool을 적용해보겠습니다.

```java
    private JedisPoolConfig jedisPoolConfig() {
        final JedisPoolConfig poolConfig = new JedisPoolConfig();
        poolConfig.setMaxTotal(128);
        poolConfig.setMaxIdle(128);
        poolConfig.setMinIdle(36);
        poolConfig.setTestOnBorrow(true);
        poolConfig.setTestOnReturn(true);
        poolConfig.setTestWhileIdle(true);
        poolConfig.setMinEvictableIdleTimeMillis(Duration.ofSeconds(60).toMillis());
        poolConfig.setTimeBetweenEvictionRunsMillis(Duration.ofSeconds(30).toMillis());
        poolConfig.setNumTestsPerEvictionRun(3);
        poolConfig.setBlockWhenExhausted(true);
        return poolConfig;
    }
```

Connection Pool 옵션을 넣고, 다시 성능 테스트를 시작해보면!

- TPS는 5.2만이 나왔습니다
- 핀포인트 결과는 평균 50ms가 나왔습니다
- 그리고 Redis의 Connection 수가 515개로 첫번째 대비 10배이상 높아졌습니다.
- Redis의 CPU 역시 3배이상 올라 69.5%를 유지했습니다.
- Connection Pool을 적용하고 TPS가 전보다 50%이상 향상되었습니다. 

그렇지만!

문제는 이보다 더 높은 TPS를 맞추려면 Redis의 CPU가 90%를 넘을수도 있습니다.

Jedis를 꼭 써야한다면 성능 테스트를 통해 적절한 Pool Size를 찾아보셔야만 합니다.

## Lettuce

Lettuce는 Netty (비동기 이벤트 기반 고성능 네트워크 프레임워크) 기반의 Redis 클라이언트입니다. 
비동기로 요청을 처리하기 때문에 고성능을 자랑합니다.

이제 테스트를 해보면!

TPS는 무려 10만을 처리합니다.

![1](/uploads/94a37dae854b278b288d8cd6909716b0/1.png)


실제로 워낙 빠르게 처리하다보니 대량의 요청을 해야하는 agent CPU가 70%까지 올라갔습니다

![2](/uploads/de8aea4121226cbccbb2858371a6259a/2.png)


핀포인트의 응답속도는 7.5ms 로 아주 고속으로 응답을 주고 있습니다.

![3](/uploads/e9a4b35cb0880a9afbc7a5ac38979f46/3.png)

그리고 Redis의 Connection은 6-7개를 유지합니다.

![1](/uploads/94a37dae854b278b288d8cd6909716b0/1.png)

CPU 역시 7%밖에 되지 않습니다.

앞에서 진행된 Jedis에 비해 압도적인 성능 차이가 보이시죠?

위에서 진행한 3개 테스트에 대한 결과입니다.

![7](/uploads/d9e783e4c1c72776d326f4218d9b9249/7.png)

Lettuce는 TPS/CPU/Connection 개수/응답속도 등 전 분야에서 우위에 있습니다.


# Lettuce를 사용합시다.

Jedis에 비해 몇배 이상의 성능과 하드웨어 자원 절약이 가능합니다. 
이외에도 몇가지 이유가 더 있는데요.

- 잘 만들어진 문서
- 공식 문서
- 깔끔하게 디자인된 코드
- Jedis의 JedisPool, JedisCluster 등의 여러 생성자들을 보면 어떤걸 써야할지 종잡을수가 없습니다.
- 빠른 피드백
- Lettuce의 경우 Issue 제기시 당일 혹은 2일안에 피드백을 남깁니다.
반면 Jedis의 경우 작년까지는 피드백을 거의 주지 않고 있었습니다
최근엔 피드백 주기가 빨라졌습니다.

원본 - https://jojoldu.tistory.com/418