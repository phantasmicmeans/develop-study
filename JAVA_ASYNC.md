# JAVA 8

### Completable Future 
대부분은 Worker Thread Pool을 따로 생성해서 비동기로 사용하면 된다. 
하지만 작업 thread가 worker thread를 기다릴 수 없을때는 위 처럼 진행하게 되면 당연히 wait 시간이 길어진다.

그래서 CompletableFuture를 자주 사용한다.
JAVA Future의 확장판이고 당연히 Future 보다는 좋은 기능을 제공한다..
그리고 Non-Blocking이다 

Future는 다음과 같은 작업이 불가능하다.

- It cannot be manually completed 
- You cannot perform further action on a Future’s result without blocking
- Multiple Futures cannot be chained together 
- You can not combine multiple Futures together 
- No Exception Handling 

CompletableFuture는 된다! 

추가로, Thread Pool을 생성하지 않아도 CompletableFuture에 의해 시작된 메소드는 ForkJoinPool.commonPool()의 thread를 사용한다.
그래도 Thread Pool 혹은 executor를 생성해서 쓰자 
