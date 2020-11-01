구성
=========
- 환경  
    - spring boot 2 / netty(boss thread = 1, worker thread = physical cpu core * 2) **default** 
        - -Dspring.cache.type=none, mustache-cache = false 
    - fetcher: api-mock, delay(50ms) 으로 대체 
      http://api-mock.naver.com/delay/50ms/one-tf/fetcher-cross.json
    - test script: http://ngrinder.navercorp.com/svn/kr19743/netty-test.py
    - pinpoint, ngrinder 
    - ncloud dev - [4vCPU, 8gb] x 1

&nbsp;

ParallelFlux에 관해 
==================
여기서 말하는 core는 **physical cpu cores** 이다. (core * 2)는 hyper threading 되어진 **logical cpu cores**를 말한다. 
###  1. ParallelFlux 
```java 
public ParallelFlux<Map<String, Object>> parallelAttributes(OneInfo info) {
    Mono<ComponentRequest> request = fetcher.createRequest(info);
    return Flux.from(request) // Flux<ComponentRequest>
               .flatMap(fetcher::receiveFlux) // Flux<ComponentReponse>
               .parallel(core * 2) // ParallelFlux<ComponentResponse>
               .runOn(Schedulers.parallel())
               .map(received -> handler.handleResponse(received, info)));
    }
```

core * 2개의 thread를 활용해 fetcher::receiveFlux() 가 리턴하는 Flux<ComponentResponse>를 병렬로 처리하고자 하였다.

결론적으로 thread pool을 활용해 publishing 되어지는 각 ComponentResponse를 handleResponse에 넘겨 병렬 처리 하고, core를 잘 활용하며 reactive 하게 처리하는 것이 의도였다.

위와 같은 코드로 부하 테스트를 실행하니 아래와 같은 결과가 나왔다.

vm | vuser | TPS | MTT (ms) | 에러율 
-- | -- | -- | -- | -- 
1 | 50 | 258 | 193.64 | 0 
1 | 100 | 258.1 | 387.17 | 0 
1 | 300 | 255.2 | 1172.4 | 0 
1 | 500 | 251.5  | 1981.41 | 0 

cpu usage는 약 30%대를 유지했다. 

### 2. Mono
&nbsp;
이 코드를 작성하기 전, 위 과정을 모두 Mono로 처리하는 코드를 작성 했었고 이는 아래와 같다. 
```java
public Mono<Map<String, Object>> attributes(OneInfo info) { 
    Mono<ComponentRequest> request = fetcher.createRequest(info);
    return Mono.from(request)
               .flatMap(fetcher::receiveMono) // Mono<List<ComponentResponse>>
               .map(received -> handler.handleResponse(received, info));
    }
```
모두 Mono로 처리하였고 사실상 reactive 하지 않다. netty를 사용하기에 적합한 코드이지도 않고 결국 thread 1개가 모두 작업을 처리하게 된다. 또한 handleResponse는 List<ComponentResponse>를 받아 for문으로 이를 처리한다. 

별 기대없이 Mono 처리에 대해 부하 테스트를 하였다.

vm | vuser | TPS | MTT (ms) | 에러율
-- | -- | -- | -- | -- 
1 | 50 | 296.6 | 168.36 | 0 
1 | 100 | 394.8 | 252.92 | 0 
1 | 300 | 526.5 | 565.14  | 0
1 | 500 | 659.3  | 754.74 | 0 | 

결과가 생각지도 못한 방향으로 나왔다. 후자의 경우 최대 2배 더 좋은 결과를 보이고 있었다. 또한 cpu usage는 80 ~ 90%에 달했다.

&nbsp;
음?
========
분명 이유가 있을것이다. 먼저 ParallelFlux.parallel() 부터 tracking 해보았다.

```java
/**
 * Prepare this {@link Flux} by dividing data on a number of 'rails' matching the
 * provided {@code parallelism} parameter, in a round-robin fashion. Note that to
 * actually perform the work in parallel, you should call {@link ParallelFlux#runOn(Scheduler)}
 * afterward.
 *
 * <p>
 * <img class="marble" src="doc-files/marbles/parallel.svg" alt="">
 *
 * @param parallelism the number of parallel rails
 *
 * @return a new {@link ParallelFlux} instance
 */
public final ParallelFlux<T> parallel(int parallelism) {
	return parallel(parallelism, Queues.SMALL_BUFFER_SIZE);
}
```

parallel(int parallelism)은 publishing 되어지는 data를 round-robin으로 parallelism 개의 '**rail**'로 나눈다. 

parallel(int parallelism)은 여러 rail을 가지는 **ParallelFlux**를 반환하며, 실제 parallel 하게 실행하려면 ParallelFlux.runOn(Scheduler)의 파라미터로 **병렬 처리가 가능한 scheduler를** 넘겨야한다. 

&nbsp;
그럼 ParallelFlux.runOn(Scheduler)를 살펴보자. 

```java
/**
 * Specifies where each 'rail' will observe its incoming values with no work-stealing
 * and default prefetch amount.
 * <p>
 * This operator uses the default prefetch size returned by {@code
 * Queues.SMALL_BUFFER_SIZE}.
 * ----- 중략 ------
 * @param scheduler the scheduler to use
 *
 * @return the new {@link ParallelFlux} instance
 */
public final ParallelFlux<T> runOn(Scheduler scheduler) {
	return runOn(scheduler, Queues.SMALL_BUFFER_SIZE);
}
``` 
**This operator uses the default prefetch size returned by {@code Queues.SMALL_BUFFER_SIZE}** 라고 한다. 한 레일당 Queues.SMALL_BUFFER_SIZE 만큼의 size를 가진다. 

Queues.SMALL_BUFFER_SIZE는 아래와 같다. 
```java
/**
 * A small default of available slots in a given container, compromise between intensive pipelines, small
 * subscribers numbers and memory use.
 */
public static final int SMALL_BUFFER_SIZE = Math.max(16,
			Integer.parseInt(System.getProperty("reactor.bufferSize.small", "256")));
```
reactor.bufferSize.small을 지정하지 않았으므로 각 rail당 16개의 size를 가질 수 있다.

사실 지금 tracking 하고자 하는 것은 각 rail의 size가 아니다. 
&nbsp;
코드를 다시 살펴보자
```java
return Flux.from(request) // Flux<ComponentRequest>
           .flatMap(fetcher::receiveFlux) // Flux<ComponentReponse>
           .parallel(core * 2) // ParallelFlux<ComponentResponse>
           .runOn(Schedulers.parallel())
           .map(received -> handler.handleResponse(received, info)));
```

fetcher::receiverFlux로 부터 publishing 되어지는 **data 들을 (core * 2)개의 rail로 나누고**, Schedulers.parallel()에 의해 병렬로 실행된다. Schedulers.parallel()은 고정 개수의 thread pool을 제공하고 default 개수는 (core * 2)이다.

&nbsp;
원인  발견!?
========
뭔가 아차 싶다..

publishing 되어지는 data를 (core * 2)개의 thread pool이 잘 나누어 처리하는 것이 아니라, **필수적으로 (core * 2)개의 rail을 생성하고** 이를 thread pool이 나누어 실행하게 될 것이다.

(core * 2)개 이상의 data가 publishing 되는 경우, (core * 2)개의 rail이 생성 되는것은 어찌보면 자연스러울 수? 있다. **문제는 publishing 되어지는 data가 (core * 2)개 이하임에도 불구하고 rail이 모두 생성될 수 있다는 점이다.**

실제 테스트를 해보았다. 

```java
return Flux.from(request) // Flux<ComponentRequest>
           .flatMap(fetcher::receiveFlux) // Flux<ComponentReponse>
           .parallel(core * 2) // ParallelFlux<ComponentResponse>
           .runOn(Schedulers.parallel())
           .log()
           .map(received -> handler.handleResponse(received, info))
           .doOnComplete(() -> log.info("complete"));
```
rail 관련 thread 확인을 위해 5번째 라인에 log(), 각 rail에 대한 처리가 완료 될 경우 "complete"를 로깅했다. 

```bash
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : request(unbounded)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : request(unbounded)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : request(unbounded)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : request(unbounded)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : request(unbounded)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : request(unbounded)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : request(unbounded)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : request(unbounded)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : request(unbounded)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : request(unbounded)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : request(unbounded)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : onSubscribe([Fuseable] FluxPublishOn.PublishOnSubscriber)
[ctor-http-nio-2] reactor.Parallel.RunOn.1                 : request(unbounded)
[     parallel-1] reactor.Parallel.RunOn.1                 : onNext(com.naver.media.one.model.ComponentResponse@517119d3)
[     parallel-2] reactor.Parallel.RunOn.1                 : onNext(com.naver.media.one.model.ComponentResponse@18b71d5)
[     parallel-3] reactor.Parallel.RunOn.1                 : onNext(com.naver.media.one.model.ComponentResponse@70ef9467)
[     parallel-2] reactor.Parallel.RunOn.1                 : onComplete()
[    parallel-10] reactor.Parallel.RunOn.1                 : onComplete()
[     parallel-6] reactor.Parallel.RunOn.1                 : onComplete()
[     parallel-9] reactor.Parallel.RunOn.1                 : onComplete()
[     parallel-7] reactor.Parallel.RunOn.1                 : onComplete()
[     parallel-5] reactor.Parallel.RunOn.1                 : onComplete()
[    parallel-10] com.naver.media.one.OneService           : complete
[     parallel-8] reactor.Parallel.RunOn.1                 : onComplete()
[     parallel-7] com.naver.media.one.OneService           : complete
[     parallel-1] reactor.Parallel.RunOn.1                 : onComplete()
[     parallel-4] reactor.Parallel.RunOn.1                 : onComplete()
[    parallel-12] reactor.Parallel.RunOn.1                 : onComplete()
[    parallel-11] reactor.Parallel.RunOn.1                 : onComplete()
[     parallel-1] com.naver.media.one.OneService           : complete
[     parallel-2] com.naver.media.one.OneService           : complete
[     parallel-4] com.naver.media.one.OneService           : complete
[     parallel-9] com.naver.media.one.OneService           : complete
[    parallel-11] com.naver.media.one.OneService           : complete
[     parallel-6] com.naver.media.one.OneService           : complete
[     parallel-5] com.naver.media.one.OneService           : complete
[     parallel-8] com.naver.media.one.OneService           : complete
[    parallel-12] com.naver.media.one.OneService           : complete
[     parallel-3] reactor.Parallel.RunOn.1                 : onComplete()
[     parallel-3] com.naver.media.one.OneService           : complete
```
12개의 rail이 생성 되었고, 12개의 thread가 이를 처리하고 있었다. publishing 되어지는 data는 3개 뿐임에도 말이다. **실제로 onNext는 3번 호출된다.**

parallel-1, 2, 3은 onNext를 통해 publishing 되어지는 data를 처리한다. 그러나 parallel-4 ~ 12는 subscribe 이후 rail 이 비어있으므로 곧바로 onComplete()를 호출하고 이후 "complete" 한다. 

Empty rail을 위해 순간적으로 12개의 모든 core를 점유 할 수 있는 상황이고, 이 부분이 가장 큰 병목지점일 것이라 생각했다. 

&nbsp;
해결
======
사실 해결법은 간단하다. fetcher::receiveFlux()의 size만큼 혹은 작은 단위로 쪼개어 rail을 생성하면 된다. 이를 통해 불필요한 rail 생성 및 그에 따른 core 점유 상황을 피할 수 있다. 

현재는 receiveFlux() 로 부터 publishing 되어지는 data는 3개 고정이다. 그러므로 3개의 rail을 만든 뒤 (core * 2)개의 thread pool을 이용해 처리해보자.

이는 항상 3개의 rail을 만들고 이후 thread pool 내에서 처리 될 것이다. 
```java
    public ParallelFlux<Map<String, Object>> parallelAttributes(OneInfo info) {
        Mono<ComponentRequest> request = fetcher.createRequest(info);
        return Flux.from(request)
                   .flatMap(fetcher::receiveFlux)
                   .parallel(3)
                   .runOn(Schedulers.parallel())
                   .map(received -> handler.handleResponse(received, info));
    }
```

### Before
vm | vuser | TPS | MTT (ms) | 에러율 
-- | -- | -- | -- | -- 
1 | 500 | 251.5  | 1981.41 | 0 

### After 
vm | vuser | TPS | MTT (ms) | 에러율
-- | -- | -- | -- | -- 
1 | 500 | 326.8  | 1524.91 | 0 

cpu usage는 40%대를 유지했다. 

처음 보다 조금 높게 측정 되었으나.. cpu usage는 여전히 낮다. 또한 Mono 처리에는 미치지 못했다. 

scheduling 변경
===========
handleResponse()에 별다른 로직이 없고, publishing data 또한 적어 병렬 처리 이점이 현재는 드러나지 않는 것 같다.  

이번에는 병렬 처리를 제외하고 subscribeOn 스케줄링을 통해 subscibing thread를 교체하는 전략을 구성했다.

<img src="https://user-images.githubusercontent.com/20153890/72679877-0301bc80-3af7-11ea-96bb-4174b4e6bd14.png" width="80%" height="350">

출처 - https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#subscribeOn-reactor.core.scheduler.Scheduler-

Run subscribe, **onSubscribe and request on a specified Scheduler's Scheduler.Worker.** As such, placing this operator anywhere in the chain will also **impact the execution context of onNext/onError/onComplete signals** from the beginning of the chain up to the next occurrence of a publishOn.

Note that if you are using an eager or blocking create(Consumer, FluxSink.OverflowStrategy) as the source, **it can lead to deadlocks due to requests piling up behind the emitter.** In such case, you should call subscribeOn(scheduler, false) instead.

subscribeOn은 위와 같은 경우에 사용하길 권장한다. 

**Typically used for slow publisher e.g., blocking IO, fast consumer(s) scenarios.**
```java
  flux.subscribeOn(Schedulers.single()).subscribe() 
```

앞서 적용했던 ParallelFlux와의 공통점은 모두 thread pool 을 통해 subscribing을 진행 한다는 점이다. 

ParallelFlux와 subscribeOn의 차이점은 다음과 같다. ParallelFlux는 **각 rail에 대한 처리를 각기 다른 thread가 진행한다(thread pool)**. 

이에 따라 각 thread가 rail 별 onNext/onError/onComplete에 대한 처리를 진행하는 모습을 위에서 확인할 수 있었다. 

그러나 subscribeOn는 **publishing thread와 subscribing thread를 분리할 뿐 어떠한 rail을 생성하진 않는다**.

즉 ParallelFlux에서 onNext/onComplete는 parallel-1, 2, 3에서 각 1번씩 실행 되었지만, subscribeOn의 경우 onNext/onComplete대해 publishing thread가 아닌 **다른 하나의 thread가 onNext/onComplete를 3번 실행할 것이다.**

```java
public Flux<Map<String, Object>> fluxAttributes(OneInfo info) {
        Mono<ComponentRequest> request = fetcher.createRequest(info);
        return Flux.from(request)
                   .flatMap(fetcher::receiveFlux)
                   .subscribeOn(Schedulers.parallel())
                   .map(received -> handler.handleResponse(received, info));
    }
```

fetcher::receiveFlux()는 약 50ms의 I/O 작업을 수행하고 있고 그에 대한 결과를 Flux로 리턴한다. 이 Flux를 subcribing 하는 경우 **onNext/onError/onComplete는 다른 thread에서 실행되어야 한다.** 

그럼 확인 해보자 

```bash
[ctor-http-nio-2] reactor.Flux.Map.2                       : onSubscribe(FluxMap.MapSubscriber)
[ctor-http-nio-2] reactor.Flux.Map.2                       : request(unbounded)
[ctor-http-nio-4] reactor.Flux.Map.2                       : onNext({ARTICLE={request={type=ARTICLE, .. 중략 .. })
[ctor-http-nio-4] reactor.Flux.Map.2                       : onNext({OFFICE_INFO={request={type=OFFICE_INFO, .. 중략 .. })
[ctor-http-nio-4] reactor.Flux.Map.2                       : onNext({OFFICE_HEADLINE={request={type=OFFICE_HEADLINE, .. 중략 .. })
[ctor-http-nio-4] reactor.Flux.Map.2                       : onComplete()
```

예상 대로 onNext/onComplete는 **reactor-http-nio-4** thread에서 분리실행 되는 모습을 확인했다.

성능 테스트 결과는 아래와 같고, ParallelFlux 처리보다 높은 성능을 볼 수 있었다. 

vm | vuser | TPS | MTT (ms) | 에러율
-- | -- | -- | -- | -- 
1 | 500 | 639.9  | 740.5 | 0 

&nbsp;

## 튜닝 결과 

최종적으로 Scheduling 변경이 가장 높은 tps 및 mtt를 보였다.

### ParallelFlux Before
vm | vuser | TPS | MTT (ms) | 에러율 | cpu usage
-- | -- | -- | -- | -- | -- 
1 | 500 | 251.5  | 1981.41 | 0 | 약 30 ~ 40 %

### ParallelFlux After 
vm | vuser | TPS | MTT (ms) | 에러율 | cpu usage
-- | -- | -- | -- | -- | -- 
1 | 500 | 326.8  | 1524.91 | 0 | 약 40 ~ 50 % 

### Mono
vm | vuser | TPS | MTT (ms) | 에러율 | cpu usage 
-- | -- | -- | -- | -- | -- 
1 | 500 | 659.3  | 754.74 | 0 | 약 80 ~ 90%

### Scheduling 변경 
vm | vuser | TPS | MTT (ms) | 에러율 | cpu usage 
-- | -- | -- | -- | -- | --
1 | 500 | 639.9  | 740.5 | 0 | 약 80 ~ 90%

## 결론
- tomcat의 경우 Mono, Scheduling 변경 처리와 성능은 비슷하나 cpu usage는 100%에 달한다. 또한 netty와 비교해 thread usage 차이는 크다. 
- 현재는 처리할 data 및 로직이 적어 Mono처리 또한 높은 성능을 나타내는 것으로  보인다. 
- 그러나 data, 그리고 cpu 연산을 필요로 하는 로직이 늘어날 수록  Scheduling 변경 버전 혹은 ParallelFlux After 버전이 더 높은 효과를 나타낼 것으로 기대한다.
