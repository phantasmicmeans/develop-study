## Observer Pattern
관찰자 패턴은 관찰자라고 불리는 자손의 리스트를 가지고 있는 **주체(Subject)** 를 필요로한다. 주체는 일반적으로 자신의 메서드 중 하나(e.g. notify)를 호출해 관찰자에게 상태 변경을 알리게 된다.

### GoF 에서의 옵저버 패턴 설명
- 객체 사이에 일대 다의 의존 관계를 정의해두어, 어떤 객체의 상태가 변할 때 그 객체의 의존성을 가진 다른 객체들이 변화를 통지받고 자동으로 갱신될 수 있게 만든다.

- 이 패턴은 이벤트 처리 기반 시스템에 필수적
- e.g. MVC 패턴, 거의 모든 UI 라이브러리가 내부적으로 이 패턴 사용

```
Subject - Observer 는 1:N 관계를 이룰 수 있음. 

Subject 
   |- Observer
   |- Observer
   |- Observer
   |- Observer 
```

일반적인 옵저버 패턴은 Subjec와 Observer 2개의 인터페이스로 구성된다. Observer는 Subject에 등록되고 Subject로 부터 알림을 하게 되는데 Subject 스스로 이벤트를 발생시키거나 다른 구성요소에 의해 호출될 수 있다. 

### Observable 예제 

**Subject / Observer interface 명세**
- subject는 이벤트 브로드캐스트용 method, 와 구독 관리 (register, unregister) method를 포함한다. 

```java
public interface Subject<T> {
    void registerObserver(Observer<T> observer);
    void unregisterObserver(Observer<T> observer);
    void notifyObservers(T event);
}
```

Observer는 이벤트를 처리하는 observe method 만 존재한다. 
```java
public interface Observer<T> {
    void observe(T event);
}
```

Observer, Subject 모두 인터페이스에 기술된 것 이상에 대해서는 서로 알지 못한다.

단순하게 Subject interface 를 구현한 ConcreteSuject를 보자.

**ConcreteSubject** (주체) 
```java
import java.util.Set;
import java.util.concurrent.CopyOnWriteArraySet;

public class ConcreteSubject implements Subject<String> {
    /**
     * multi-thread 안정성 유지를 위해 업데이트시마다 새 복사본을 생성하는 Set 구현체 사용
     * 복사 비용은 큼 -> 구독자 목록 변경은 거의 없음.
     */
    private final Set<Observer<String>> observers = new CopyOnWriteArraySet<>();

    @Override
    public void registerObserver(Observer<String> observer) {
        observers.add(observer); // register observer
    }

    @Override
    public void unregisterObserver(Observer<String> observer) {
        observers.remove(observer); // remove observer
    } 

    @Override
    public void notifyObservers(String event) {
        observers.forEach(observers -> observers.observe(event)); // notify observer
    }
}
```

그리고 Observer interface 를 구현한 ConcreteObserverA, B 클래스를 생성하자. 이들은 단순히 브로드캐스트 받은 이벤트 처리용 메소드인 observe(event) method를 구현한다.

```java
public class ConcreteObserverA implements Observer<String> {
    @Override
    public void observe(String event) {
        System.out.println("Observer A:" + event);
    }
}

public class ConcreteObserverB implements Observer<String> {
    @Override
    public void observe(String event) {
        System.out.println("Observer B:" + event);
    }
}

```
단순히 observer 들을 등록 / 해지할 register, unregister method와, 모든 observer 에게 이벤트 브로드캐스트용 notifyObserver method를 포함한다.

테스트 코드는 아래와 같다. 

```java
    @Test
    public void observerHandleEventsFromSubject() {
        // given
        Subject<String> subject = new ConcreteSubject();
        Observer<String> observerA = new ConcreteObserverA();
        Observer<String> observerB = new ConcreteObserverB();

        // when
        subject.notifyObservers("no subscriber!");
        subject.registerObserver(observerA);
        subject.registerObserver(observerB);

        subject.notifyObservers("event !");

        subject.unregisterObserver(observerB);

        subject.notifyObservers("event for A");
        subject.unregisterObserver(observerA);

        subject.notifyObservers("no subscriber");
    }
```

또한 아래처럼 람다로 observer를 작성해도 된다. 

```java
    @Test
    public void subjectLeverageLambdas() {
        Subject<String> subject = new ConcreteSubject();
        subject.registerObserver(new Observer<String>() {
            @Override
            public void observe(String event) {
                System.out.println("A : " + event);
            }
        });
        subject.registerObserver(event -> System.out.println("B : " + event));
        subject.notifyObservers("event start for A & B");
    }
```

ConcreteSubject는 주 스레드에서 observer들에게 이벤트 브로드캐스팅을 하고 있다. 따라서 이벤트 처리 시간이 긴 Observer 존재시 처리 시간을 늘어나게 된다. 

아래 예시는 A Observer 의 경우 이벤트 처리를 위해 3s 이상의 시간을 필요로하고, B Observer 의 경우 이벤트 처리 시간이 빠른 경우를 나타낸다. 

```java

    @Test
    public void subjectThreadTest() {
        Subject<String> subject = new ConcreteSubject();
        subject.registerObserver(event -> { // A observer 는 이벤트 처리시간이 길다. 
            try {
                Thread.sleep(3000);
            } catch (Exception e) {

            }

            System.out.println(Thread.currentThread().getName());
            System.out.println("A: " + event);
        });

        subject.registerObserver(event -> {
            System.out.println(Thread.currentThread().getName());
            System.out.println("B: " + event);
        });

        subject.notifyObservers("event!");

        try {
            Thread.sleep(3100);
        } catch (Exception e) {

        }
    }
```

이벤트 처리시간이 상당히 긴 Observer가 존재할 수 있으므로 Observer 가 많을 경우 추가적으로 스레드풀을 할당해야한다. (실제 구현시는 검증된 라이브러리를 사용하자)

병렬로 이벤트를 브로드캐스팅 하기 위한 ConcreteSubject 및 테스트 코드는 아래와 같다.

```java
public class ConcreteParallelSubject implements Subject<String> {
    private final ExecutorService ex;

    public ConcreteParallelSubject(ExecutorService ex){
        this.ex = ex;
    }
    /**
     * multi-thread 안정성 유지를 위해 업데이트시마다 새 복사본을 생성하는 Set 구현체 사용
     * 복사 비용은 큼 -> 구독자 목록 변경은 거의 없음.
     */
    private final Set<Observer<String>> observers = new CopyOnWriteArraySet<>();


    @Override
    public void registerObserver(Observer<String> observer) {
        observers.add(observer);
    }

    @Override
    public void unregisterObserver(Observer<String> observer) {
        observers.remove(observer);
    }

    @Override
    public void notifyObservers(String event) {
        observers.forEach(observers -> {
            ex.submit(() -> observers.observe(event));
        });
    }
}
```
```java
    @Test
    public void subjectParallelTest() throws InterruptedException {
        final ExecutorService ex = Executors.newFixedThreadPool(10);

        Subject<String> parallelSubject = new ConcreteParallelSubject(ex);
        parallelSubject.registerObserver(event -> { // 이벤트 처리 시간이 길다. 
            try {
                Thread.sleep(3000);
            } catch (Exception e) {

            }

            System.out.println(Thread.currentThread().getName()); // thread name 
            System.out.println("A : " + event); 
        });

        parallelSubject.registerObserver(event -> {
            System.out.println(Thread.currentThread().getName());
            System.out.println("B : " + event);
        });

        parallelSubject.notifyObservers("event!");

        ex.awaitTermination(3100, TimeUnit.MILLISECONDS); // 모든 이벤트가 종료될때까지 기다린다. 
    }
```


여기까지가 옵저버 패턴에 대한 설명과 간단한 구현이다.

## Observer Pattern VS Publisher-Subscribe Pattern

옵저버 패턴과 Publisher-subscriber (발행 -구독)패턴은 약간 다르다. 발행-구독 패턴은 '이벤트 채널'(메시지 브로커 or 이벤트 버스) 이라는 간접적인 계층을 하나 더 포함한다. 구독자는 이벤트를 브로드캐스트하는 '이벤트 채널'을 알고 있지만 이벤트 게시자가 누구인지는 신경쓰지 않는다.

e.g. 토픽 기반 시스템(카프카)
