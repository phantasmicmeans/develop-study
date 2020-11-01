## Reactive Streams

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

앞서 살펴봤던 Observable은 
