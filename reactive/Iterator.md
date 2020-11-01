## Iterator 반복자 패턴
- RxJava(ReactiveX)는 Observer pattern, Iterator pattern 및 함수형 프로그래밍의 조합으로 사용된다.

[Observer](#https://github.com/phantasmicmeans/develop-study/blob/master/reactive/observer.md) pattern은 무한한 데이터 스트림에 매력적이다. 
그러나 데이터 스트림의 '끝' 을 알리는 기능이 없고, '에러' 전파 메커니즘 또한 포함하고 있지 않다. 또한 consumer가 준비되기 전 producer가 이벤트를 생성하는 것은 그다지 좋은 상황도 아니다. 

동기식의 경우 이런 때를 대비해서 Iterator 패턴이 존재한다. 

```java
public interface Iterator<T> {
    T next();
    boolean hasNext();
}
```

위 Iterator interface는 하나씩 항목을 검사하기 위한 next() method와 다음 항목의 존재 여부를 위한 hasNext() method를 선언한다. 
이는 스트림의 '끝'을 나타내는 역할을 한다.

## Observer pattern + Iterator Pattern = Reactive Stream

- Iterator로 스트림의 '끝'을 나타내고, Observer pattern의 async한 이벤트 실행을 결합하면 아래와 같을 수 있다.

```java
public interface RxObserver<T> {
    void onNext(T next);
    void onComplete();
    void onError(Exception e);
}
``` 

RxObserver는 Iterator와 비슷하면서도 조금 다르다. next() 대신 onNext(T next) 콜백에 의해 RxObserver에 이벤트가 통지된다. 
또한 hasNext() 대신 스트림의 '끝' 을 나타내는 onComplete() method를 가진다.

Iterator는 next()를 처리하면서 error 를 발생시킬 수 있다. 이 error는 RxObserver로 전파되어야 처리가 용이할 것이다.
이를 위해 onError() 콜백도 추가한다.

위 인터페이스는 리액티브 스트림에서의 모든 컴포넌트 사이에 데이터가 흐르는 방법을 정의한다. 
