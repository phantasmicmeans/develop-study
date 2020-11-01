## Duality, Reactive Streams

앞선 글에서 Observer pattern + Iterator 패턴(이벤트의 `끝`, 그리고 `Error`) == Reactive Stream이라고 정리했다. 

이와 더불어 토비의 리액티브 스트림을 보고 Iterable <-> Observable (Duality) 에 대해 정리해보겠다.

## Duality

Iterable | Observable
-------- | ----------
[Iterable] | [Observable]
[Pull] | [Push]
[값을 끌어온다는 의미] | [값을 가져가라는 의미]
[iterator.next()] | [notifyObservers(arg)]


## Iterable, Observable Pull, Push 
먼저 Iterable과 Observable의 Pull-Push 차이를 보자. Iterable을 구현한 클래스들은 for-each statement를 사용 가능하다.

```java
public interface Iterable<T> {
    /**
     * Returns an iterator over elements of type {@code T}.
     *
     * @return an Iterator.
     */
    Iterator<T> iterator();
    ...
}
```

Iterable은 단순히 `Iterator<T> iterator()`를 선언하고 있고 이를 구현하면 된다. 
Iterator는 앞선 글에서처럼 hasNext(), next()를 선언하고 있다.

```java
public interface Iterator<E> {
    /**
     * Returns {@code true} if the iteration has more elements.
     * (In other words, returns {@code true} if {@link #next} would
     * return an element rather than throwing an exception.)
     *
     * @return {@code true} if the iteration has more elements
     */
    boolean hasNext();

    /**
     * Returns the next element in the iteration.
     *
     * @return the next element in the iteration
     * @throws NoSuchElementException if the iteration has no more elements
     */
    E next();
    ...
}
```

앞서 살펴봤던 Observable은 Observer들을 등록하고, `notifyObserver()`를 통해 Observer들에게 이벤트를 push하는 방식이였지만, Iterable의 경우 Iterator를 통해(next()) 다음 이벤트?를 호출한다. 그렇기에 Observable의 경우 이벤트 주체가 이벤트를 Push, Iterable의 경우 이벤트 수신자가 이벤트를 Pull한다고 볼 수 있는것이다.

코드로 보면 아래와 같을 수 있다. 

**Iterable - Pull**
```java
  Iterable<Integer> iterable = () -> { 
      new Iterator<Integer>() { // iterator 구현 
              int start = 0; // 1~10 까지 event 방출
              int MAX = 10;

              @Override
              public boolean hasNext() {
                 return this.start < MAX;
              }

              @Override
              public Integer next() {
                  return this.start++;
              }
          };
      };

  for (Iterator<Integer> iterator = iterable.iterator(); iterator.hasNext();) {
      System.out.println(iterator.next()); // event pull 
  }

  for (Integer i : iterable) { // 위와 동일 
      System.out.println(i);
  }
```

**Observable - Push**
```java
    @Test
    public void ObservableTest() throws InterruptedException {
        ExecutorService ex = Executors.newFixedThreadPool(1);
        RxObservable<String> observable = new ConcreteRxObservable(ex);
        RxObserver<String> observer = new RxObserver<String>() {
            @Override
            public void onNext(String next) {
                System.out.println("RxObserver B");
                System.out.println(Thread.currentThread().getName());
                System.out.println("RxObserver B " + next);
            }

            @Override
            public void onComplete() {
                System.out.println("RxObserver B Complete");
            }

            @Override
            public void onError(Exception e) {

            }
        };

        observable.registerObserver(observer);
        observable.notifyObservers("Event!"); // event push 
        ex.awaitTermination(1000, TimeUnit.MILLISECONDS);
    }
```

결론은 Iterable <-> Observable 간에는 Duality가 존재하고, 이는 event Push(Observable#notifyObservers) / Pull(Iterable#next)의 차이이다.
