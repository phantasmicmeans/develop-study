https://micrometer.io/docs/concepts

## 1. Registry 
Meter is the interface for collecting a set of measurements (which we individually call metrics) about your application. 
Micrometer에서 Meter는 `MeterRegistry`로 부터 만들어진다. 이를 서포트하는 모든 모니터링 시스템(ex. prometheus)들은 MeterRegistry를 구현하여 개발된다.

Micrometer에서의 `SimpleMeterRegistry`는 각 meter의 최근 value들을 메모리에 들고 있다. 아직 무언가 모니터링 시스템이 없다면 아래처럼 생성해도 된다.
```java
MeterRegistry registry = new SimpleMeterRegistry();
```
---
**NOTE**

A SimpleMeterRegistry is autowired for you in Spring-based apps.

---

## 2. Composite Regisitries
Micrometer는 하나의 모니터링 시스템에서 여러 meteric을 동시에 보여주기 위한 용도로, CompositeMeterRegistry라는 multiple registry를 지원한다.
```java
CompositeMeterRegistry composite = new CompositeMeterRegistry();

Counter compositeCounter = composite.counter("counter");
compositeCounter.increment(); (1)

SimpleMeterRegistry simple = new SimpleMeterRegistry();
composite.add(simple); (2)

compositeCounter.increment(); (3)
```

1. Increments are NOOPd until there is a registry in the composite. The counter’s count will still yield 0 at this point.
2. A counter named "counter" is registered to the simple registry.
3. The simple registry counter is incremented, along with counters for any other registries in the composite.


## 3. Global Registry
Micrometer는 static한 global registry인 ```Metrics.globalRegistry```를 제공한다. 이 globalRegistry 또한 Composite Registry 이다. 

```java
	public Object gaugeRegistry(@PathVariable String gauge) {
		Metrics.gauge("firstListSize", 10);
		Metrics.gauge("listSize", Tags.empty(), new ArrayList<>(), List::size);

		return Metrics.globalRegistry.get(gauge).gauge().measure();
```

## 4. Meter
Meter는 다음과 같은 컴포넌트를 포함한다.

- Gauge
- Counter
- Timer
- DistributionSummary
- LongTaskTimer
- TimeGauge
- FunctionCounter
- FunctionTimer

각 Meter는 name과 dimension에 의해 uniquely identified 되는데, 일반적으로 tag가 짧기에 사용한다. 
     
### 4.1 Tag Naming
태그 이름을 지정할 때 meter 이름에 대해 설명하는 동일한 소문자 닷 표기법 따르는 것이 좋다. 또한 Tag의 value는 null이면 안된다. 

만일 number of http requests and the number of database calls를 측정한다고 하면 아래와 같을 수 있다.

**Recommended approach**

```
registry.counter("database.calls", "db", "users")
registry.counter("http.requests", "uri", "/api/users")
```


## 5. Counter
Counter는 단일 메트릭 (개수)를 report 하는데 쓰인다. 양수의 fixed 된 값을 증가시킬 수 있다. 
- Counters report a single metric, a count.
- The Counter interface allows you to increment by a fixed amount, which must be positive

---
**Tip**

- 시간측정 혹은 summarize 목적으로 사용하지 말아라. 
- Never count something you can time with a `Timer` or summarize with a `DistrubutionSummary`

---
카운터에서 그래프와 경고를 생성할 때, 일반적으로 주어진 시간 간격 동안 일부 이벤트가 발생하는 속도를 측정하는 데 가장 관심을 기울여야 합니다.
단순 큐를 고려해봐라. 카운터는 항목이 삽입 및 제거되는 속도와 같은 항목을 측정하는 데 사용할 수 있다.

처음에는 비율보다는 절대 카운트를 시각화하는 것을 상상하는 것은 유혹적이다. 그러나 절대 카운트는 일반적으로 어떤 것이 사용되는 신속성의 함수이며, 응용 프로그램의 수명도 계측된다.
일정 시간 간격마다 대시보드와 카운터 속도에 대한 경고를 작성하면 앱의 수명이 무시되므로 앱이 시작된 지 한참 후에 비정상적인 동작을 볼 수 있습니다.

타이머는 타이밍에 들어가는 메트릭 세트의 일부로 타이밍 설정 이벤트 수를 기록하므로 카운터를 사용하기 전에 타이머 섹션을 반드시 읽어야 합니다.
시간을 기록하려는 코드 조각의 경우 카운터를 별도로 추가할 필요가 없습니다.

The following code simulates a real counter whose rate exhibits some perturbation over a short time window.

```java
Normal rand = ...; // a random generator

MeterRegistry registry = ...
Counter counter = registry.counter("counter"); (1)

Flux.interval(Duration.ofMillis(10))
        .doOnEach(d -> {
            if (rand.nextDouble() + 0.1 > 0) { (2)
                counter.increment(); (3)
            }
        })
        .blockLast();
```

1. Most counters can be created off of the registry itself with a name and, optionally, a set of tags. (1)
2. A slightly positively-biased random walk. (2) 
3. This is how you interact with a counter. You could also call counter.increment(n) to increment by more than 1 in a single operation. (3)

There is also a fluent builder for counters on the Counter interface itself, providing access to less frequently used options like base units and description. You can register the counter as the last step of its construction by calling register.

```java
Counter counter = Counter
    .builder("counter")
    .baseUnit("beans") // optional
    .description("a description of what this counter does") // optional
    .tags("region", "test") // optional
    .register(registry);
```

## 6. Gauge
gauge는 현재 value를 핸들링하는데 적절하다. 예를들면 컬렉션 혹은 맵의 사이즈, 러닝 상태의 스레드 개수 등
- A gauge is a handle to get the current value.
- Typical examples for gauges would be the size of a collection or map or number of threads in a running state.

---
**Tip**

- Gauge는 bound가 있는 경우 사용하도록한다. request count를 모니터링하는 것은 bound가 존재하지 않아 위험하다. (Counter를 알아봐라)
- Gauges are usefule for monitoring things with natural upper bounds. 
- We don't recommend using a gauge to monitor things like `request count`
- As they can grow without bound for the duration of an application instance's life.
- **Never gauge something you can count with a `Counter`.

---

