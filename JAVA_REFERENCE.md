Strong Reference & Weak Reference
==============================
JVM 내의 Reference란 무엇인지에 대해 먼저 이해해야 하며, 이를 GC와 관련해 이해하는 것이 좋다.

## JVM Heap Memory Reference 
JVM의 Garbage Collector가 Garbage Collecting을 행하는 기준을 설명한다.

일반적으로 Garbage Collector는 Stack내의 원소들을 훑으며 각 원소가 Heap Memory내의 어떠한 객체를 참조하고 있는지 체크하고 기록한다. 일반적인 경우 Stack내의 원소를  객체 참조의 Root Set이라 한다. 물론 이것만 있는 것은 아니다. 결론적으로 이를 Marking이라고 한다.

Heap의 객체들에 대해 참조는 다음과 같다.
1. 힙 내의 다른 객체에 의한 참조
2. Java Method 실행시 사용되는 지역변수 및 파라미터에 의한 참조
3. JNI에 의해 생성된 객체에 대한 참조
4. Method 영역의 static 변수에 의한 참조

**이들 중 객체 참조의 Root Set이라 부를 수 있는 것을 1을 제외한 나머지이다.**

**즉 Marking이란 객체 참조의 Root Set 원소를들을 탐색하며 각 원소가 Heap Memory내의 특정 객체를 참조하고 있는지 판단하는 행위를 말하며 아래처럼 나타낼 수 있다.**

![](https://d2.naver.com/content/images/2015/06/helloworld-329631-2.png)

출처: https://d2.naver.com/helloworld/329631

&nbsp;

Marking에 의해 Heap Memory 내에는 체크되지 않은 객체들이 존재하게 되는데, GC는 이들을 제거한다. 이 과정을 Sweep이라 한다.
즉 기본적인 GC의 원리는 **Mark & Sweep** 이며 이를 **stop the world** 라고 한다. GC과정에서는 모든 스레드가 일시적으로 멈추게된다.

GC의 기본원리는 위와 같다. 조금 더 상세하게 들어가면 Young 영역, Old 영역으로 나뉘며 Young 영역을 GC하는 행위를 Minor GC라 하며, Old 영역을 비우는 행위를 Major GC라 한다. 

결론적으로 GC를 행할 때 stop the world가 발생하게 되어 모든 스레드가 일시 정지하게 된다.
이 시간을 각 상황에 맞게 튜닝하는 작업은 GC 튜닝이라고 한다.

## Strong Reference

위에서 **객체 참조의 Root Set**에 대해 설명하였다. Heap내의 다른 객체에 의해 참조되어진 객체일지라도 Root Set이 존재하면 이는 Reachable하며 GC의 대상이 아니다.

그러나 Heap내의 다른 객체에 의해 참조되어진 **객체가 Root Set에 의해 참조되어지고 있지 않다면** 이는 **Unreachable** 이며 GC의 대상이다.

이는 java에서 말하는 일반적인 참조이며 이를 흔히 Strong Reference라고 한다.

A strong reference is an ordinary Java Reference.
```java
    StringBuffer buffer = new StringBuffer();
``` 
위 코드는 new 연산자에 의해 StringBuffer 객체를 생성해냈다. 이때 buffer라는 변수는 stack 내에 존재하게 되며,  new StringBuffer()에 의해 생성된 객체를 참조하게 된다. 이를 우리는 Reachable하다고 하며, Strong Reference 되어지고 있다고 말한다.

scope를 가지는 경우도 마찬가지이다. 어떠한 메소드 내에서 위의 코드를 똑같이 적용하면 아래와 같다.

```java
public void doSomething() {
	StringBuffer buffer = new StringBuffer(); 
	buffer.append(“A”);
	buffer.append(“B”);
	
	System.out.println(buffer.toString());
	return;
}
```
위와 같은 scope내의 buffer는 return 전까지 Root Set이 존재하는 Strong Referenced 객체이다.
이후 “return”시 메소드를 종료하며 참조를 해제해제하며 Unreachable 되어지고 GC의 대상이 된다.  

## Weak Reference 

Weakly Referenced Object
**A weakly referenced object is cleard by the Garbage Collector when it's weakly reachable.**
