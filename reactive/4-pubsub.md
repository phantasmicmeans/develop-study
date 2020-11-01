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
Observable이 제공할 데이터가 많든, 이벤트를 받은 Observer의 처리시간이 길든 신경쓰지 않고 말이다. 이러한 경우는 backpressure 조절이 쉽지않다. 
중간에 버퍼용 큐를 두고 publishing 데이터를 queue에 offer하고 subscriber는 poll하여 사용할 순 있겠지만, 
Publisher의 publishing 속도가 매우 빠른경우 혹은 버퍼 큐의 메모리 문제가 발생할 수 있고, 소실될 수도 있다.

다시 돌아와 Subscriber가 Publisher와 상호 작용하기 위한 중재자가 바로 Subscription 이다. Observable은 event Push 방식으로 모든 Observer에게 이벤트를 전파했었다.

그러나 Reactive Stream은 기본적으로 event pull 방식으로서 유연하게 backpressure 처리가 가능해진다. 이때 사용할 정보가 Subscription인데 아래와 같은 명세로 이루어져있다.

```java
package org.reactivestreams;

public interface Subscription {
    void request(long var1);

    void cancel();
}
```

Subscriber는 `Subscription.request(long n)`을 통해 이벤트를 요청하고, Publisher는 직접 Subscriber에게 onNext() 호출을 통해 event를 전달한다. 
이 말은 Subscriber는 `Subscription.request(long n)` signal을 demand 해야만 onNext() signal을 받을 수 있다 는 얘기이다. 
추가로 subscriber가 처리할 수 있을 만큼의 request(n)를 하여 pull하는데 이를 dynamic pull 방식이라 하고 이것이 바로 Backpressure의 기본 원리이다.

이때 중요한 것은 Subscriber는 Subscription을 통해 request 요청을 하고, Publisher는 request 요청을 받아 Subscriber에게 직접 onNext()를 호출한다.


<img src="https://user-images.githubusercontent.com/20153890/97805665-ece67680-1c9a-11eb-98b0-f78d612d6c49.png" width=500>

예제 코드는 아래와 같다. 

```java
        Stream<Integer> integerStream = IntStream.range(0, 100).boxed();

        Publisher<Integer> publisher = new Publisher<Integer>() {
            Iterator<Integer> iterator = integerStream.iterator();

            @Override
            public void subscribe(Subscriber<? super Integer> subscriber) {
                subscriber.onSubscribe(new Subscription() {
                    @Override
                    public void request(long l) {
                        if (!iterator.hasNext()) {
                            subscriber.onComplete();
                        }

                        while (l-- > 0 && iterator.hasNext()) {
                            System.out.println("[Publisher] : " + Thread.currentThread().getName());
                            subscriber.onNext(iterator.next());
                        }
                    }

                    @Override
                    public void cancel() {

                    }
                });
            }
        };

        Subscriber<Integer> subscriber = new Subscriber<Integer>() {
            private Subscription subscription;
            @Override
            public void onSubscribe(Subscription subscription) {
                this.subscription = subscription;
                this.subscription.request(1);
            }

            @Override
            public void onNext(Integer integer) {
                System.out.println("[Subscriber] : " + Thread.currentThread().getName() + " event : " + integer);
                this.subscription.request(1);
            }

            @Override
            public void onError(Throwable throwable) {
            }

            @Override
            public void onComplete() {
                System.out.println("onComplete");
            }
        };

        publisher.subscribe(subscriber);
```
