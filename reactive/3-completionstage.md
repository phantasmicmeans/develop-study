## 2. 알고 넘어가자 CompletableFuture! <a name="pre-completable"></a>


CompletableFuture는 다음과 같은 클래스 선언을 가집니다.

```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> 
``` 

비동기 task의 결과를 받아볼 수 있는 Future, 그리고 또 다른 하나인 CompletionStage를 구현하고 있습니다. Future는 java 개발자라면 한번쯤은 경험해보셨을 내용이지만 CompletionStage는 익숙하지 않으실 수 있으니 짧게 설명 드리고 넘어가겠습니다.

[CompletionStage](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html)의 명세에는 다음과 같이 쓰여 있습니다.

`A stage of a possibly asynchronous computation, that performs an action or computes a value when another CompletionStage completes.`

해석과는 약간 다를 수 있으나 `어떤 비동기식 stage가 존재하고 해당 stage에 의존적으로 (완료될 때) 이후 수행할 stage를 정한다` 라고 받아들이면 쉽게 이해 될 것 같습니다.

결론적으로 **CompletableFuture는 Future와 CompletionStage를 모두 구현**하고 있으므로 비동기 task의 결과를 가지고 여러 stage를 만들어 다른 task (async, sync 모두) 를 손쉽게 실행할 수 있는데요.

CompletableFuture를 잘 활용하면 기존 다른 Future 계열 (listenableFuture 등, callback hell) 보다 더 우아하게 처리할 수 있습니다. 

```java
CompletableFuture.supplyAsync(() -> newsStatApiClient.get(list))
      .thenApply(newsStatResult -> newsVodApiClient.post(newsStatResult))
      .thenAccept(result -> log.info(result));
```

위 코드는 3개의 stage가 존재하며 모두 이전 stage에 종속적이게 됩니다. 

**추가로 java8에 등장한 [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)는 Functional Interface와 일맥상통합니다.**

CompletableFuture를 활용하다보면 대표적으로 `supplyAsync~`,  `thenApply~`, `thenAccept~` 형태를 자주 보실 수 있을텐데요, 아래처럼 모두 Functional Interface를 활용합니다.

- `supplyAsync~` 의 경우 첫 stage에서 비동기 task를 supply 해야하므로  `Supplier interface` 를 인자로 받는 정적 팩토리 메소드
```java
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
```

- `thenApply~` 의 경우 이전 stage의 결과를 가지고 다음 stage에 제공할 결과를 반환해야하므로 `Function interface`를 인자로
```java
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
```

- `thenAccept~` 의 경우 이전 stage의 결과를 소비하는 `Consumer interface`
```java
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action) 
```

서론이 길어졌습니다만 결론적으로 **Functional Interface를 통해 람다식을 작성할 수 있고, 함수형 스타일로 조금 더 기존 코드보다 깔끔한 코드를 작성할 수 있습니다.**
