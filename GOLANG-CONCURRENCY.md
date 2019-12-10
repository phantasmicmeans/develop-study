goroutine이란 
===============

Go루틴은 Go Runtime 이 관리하는 Logical Thread이다. Go에서 "go" 키워드를 사용해 함수를 호출하면, Runtime시 새로운 goroutine을 실행한다.

goroutine은 asynchronusly하게 routine을 실행한다. 따라서 여러 코드를 **concurrent**하게 실행하는데 사용된다.

goroutine은 OS Thread 보다 훨씬 가볍게 Async Cocurrent 처리를 구현하기 위해 만든 것이다. 이는 기본적으로 Go Runtime이 자체 관리한다.

Go Runtime 상에서 관리된느 작업단위인 여러 goroutine은 종종 하나의 OS Thread 1개로도 실행되곤 한다.

즉, goroutine은 OS Thread와 1대1로 매핑되지 않고, Multiplexing으로 훨씬 적은 OS Thread를 사용한다. 

Memory측면에서도 일반 OS Thread가 1MB의 stack을 갖는 반면, goroutine은 이보다 훨씬 작은 몇 KB의 stack을 갖는다고 한다. 또한 이는 필요시 동적으로 증가한다. 

**Go Runtime은 goroutine을 관리하면서 Go Channel을 통해 goroutine간 통신을 쉽게 할 수 있도록 한다.**

## Multi CPU Core 처리 
Go는 default로 1개의 CPU 사용. 여러개의 goroutine을 만들더라도 1개의 CPU에서 작업을 시분할하여 Concurrent하게 처리한다.

만약 machine이 복수의 CPU 보유시, Go 프로그램을 Parallel하게 할 수 있다. 아래와 같이 하면 된다.

```golang 
package main

func main() {
    runtime.GOMAXPROCS(4) // 4개의 CPU 사용 
}
``` 

주의할 점 
- **Concurrecy와 Paralleisim은 다르다.**


## Go Channel
Go Channel은 그 채널을 통해 data를 송수신하는 통로라고 보면 된다. 채널은 make() 함수를 통해 미리 생성되어야 하며, Channel 연산자 <-을 통해 data를 보내고 받는다. 

Channel은 흔히 goroutine들 사이에서 data를 주고 받는데 사용되는데, 상대편이 준비될 때까지 channel에서 대기함으로써 별도의 lock을 걸지 않고 data를 동기화하는데 사용된다.

아래 예제는 int channel을 생성하고, 한 goroutine에서 그 channel에 123이란 정수 데이터를 보낸 후 main routine에서 데이터를 다시 받는 코드이다.

**channel 생성 주의사항**
- make() 함수에 type을 지정해 주어야 한다. ex. make(chan int)
- channel로 data를 보내는 경우 [채널명 <- data]
- channel로 부터 data를 수신하는 경우 [<- 채널명]

```go
package main

func main() {
    ch := make(chan int) // create int channel 
    go func() {
        ch <- 123 // 채널에 123 송신 
    }()

    var i int
    i = <- ch // 채널로부터 123 수신 
    println(i)
}
```

Go Channel은 수신자와 송신자가 서로를 기다리는 속성을 가진다. 이를 이용하여 goroutine이 끝날 때까지 기다리는 기능을 구현할 수 있다.

즉, 익명함수를 사용한 goroutine에서 어떤 작업이 실행되고 있을 때, main routine은 <- done에서 계속 수신하며 대기하고 있게 된다. 익명함수의 goroutine이 끝난 후, done channel에 true를 보내면 수신자 main routine은 이를 받고 종료한다.

```go
package main
import "fmt"
func main() {
    done := make(chan bool)
    go func() {
        for i := 0; i < 10; i ++ {
            fmt.Println(i)
        }
        done <- true
    }()

    // 위 goroutine이 끝날 때까지 대기 
    <- done
}
```

## Go Channel Buffering 
Go는 2가지 Channel을 가진다.
- Unbuffered Channel
- Buffered Channel

현재까지 위의 예제는 Unbuffered Channel로, **이 채널은 하나의 수신자가 데이터를 받을 때까지 송신자가 데이터를 보내는 채널에 묶여있게 된다.**

Buffered Channel은 수신자가 받을 준비가 되어있지 않더라고, 지정된 buffer size만큼 buffer를 전송하고 계속 다른일을 수행할 수 있다.

Buffer Channel은 아래와 같은 형태를 지니는데, **2번쨰 param N에 buffer 개수**를 넣어야한다.  
```go 
    make(chan type, N)
```

예를 들어 make(chan int, 10) 은 10개의 int형을 갖는 Buffer Channel를 만든다.

Buffer Channel을 이용하지 않는 경우, 아래와 같은 코드는 Dead Lock Error를 일으킨다. main routine에서 channel에 1을 보내며 상대편 수신자를 기다리고 있는데, 이 channel을 받는 수신자 goroutine이 없기 때문이다. 

```go
package main
import "fmt"
func main() {
  c := make(chan int)
  c <- 1   //수신루틴이 없으므로 데드락 
  fmt.Println(<-c) //코멘트해도 데드락 (별도의 Go루틴없기 때문)
}
```

하지만 아래처럼 Buffer Channel를 사용하면 수신 goroutine이 없더라고 최대 버퍼 수까지 송신 가능하다.

```go
package main
import "fmt"
func main() {
    ch := make(chan int, 1)
    //수신자가 없더라도 보낼 수 있다.
    ch <- 101
    fmt.Println(<-ch)
}
```

여기까지가 Go Routine 및 Channel에 대한 간단한 설명이고 더 깊은 내용은 코드를 통해 알아본다. 
