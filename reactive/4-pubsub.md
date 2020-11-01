## Publisher 

Rective Stream에서 정의하는 Publisher는 sequential한 element들을 제공하는 provider이고, Subscriber의 요구에 따라 publish 한다.
앞서 살펴보았던 Observable과 동등한 의미로 보면 된다.

Observable은 `registerObserver(Observer<T> observer)`로 subscriber를 등록했다면, Publisher는 `subcribe(Subscriber<? super T> subscrbier)`를 활용해
Subscriber를 정한다.

 기준 | Observable | Publisher
---------- | --------- | ---------
Provider | O  | O
Receiver | Observer | Subscriber
Add Receiver | Observable.addObserver(ob) | Publisher.subscribe(sb) 

따라서 Reactive Stream의 Publisher 명세를 보면 아래와 같다.

```java
package org.reactivestreams;

public interface Publisher<T> {
    void subscribe(Subscriber<? super T> var1);
}
``` 

앞서 설명했던 아래의 RxObservable과 대조하면, 단순히 registerObserver에 대등되는 `subscribe(Subscriber<? super T> var1)`만을 선언한다. 

```java
public interface RxObservable<T> {
    void registerObserver(RxObserver<T> observer);
    void unregisterObserver(RxObserver<T> observer);
    void notifyObservers(T event);
    void notifyComplete();
}
```

## Subscriber

Subscriber는 아래와 같은 명세를 가진다. 앞서 살펴봤던 RxObserver와 거의 동일하나 publisher의 subscribe()의 callback인 onSubscribe를 포함한다.
또 다른 특이점은 인자로 `Subscription`을 받고 있는 것이다. 

```java
package org.reactivestreams;

public interface Subscriber<T> {
    void onSubscribe(Subscription var1);

    void onNext(T var1);

    void onError(Throwable var1);

    void onComplete();
}
```

reactive streams는 크게 아래 4가지의 interface로 이루어져 있다. 그중 하나인 Subscription은 backpressure 그 자체라고도 표현할 수 있을 것 같다. 

```
org.reactivestreams
- Subscription
- Subscriber
- Publisher
- Processor
```

기존 Observable에서는 단순히 `notifyObservers(T event)` 메소드를 통해 모든 옵저버에게 이벤트를 알렸다. 
Observable이 제공할 데이터가 많든, 이벤트를 받은 Observer의 처리시간이 길든 신경쓰지 않고 말이다.

Subscriber가 Publisher와 상호 작용하기 위한 중재자가 바로 Subscription 이다. 
Observable은 event Push 방식으로 모든 Observer에게 이벤트를 전파했었다.

그러나 Reactive Stream은 기본적으로 event pull 방식으로서 조금 더 유연하게 backpressure 처리가 가능해진다.
이때 사용할 정보가 Subscription인데 아래와 같은 명세로 이루어져있다.

Subscriber는 Subscription을 토해 이벤트를 요청하고, Publisher는 직접 Subscriber에게 event를 전달한다. 
이때 중요한 것은 Subscriber는 Subscription을 통해 request 요청을 하고, Publisher는 request 요청을 받아 Subscriber에게 직접 onNext()를 호출한다.

```java

package org.reactivestreams;

public interface Subscription {
    void request(long var1);

    void cancel();
}
```
