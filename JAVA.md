# JAVA 8

### Java.util.function 패키지 탐색
- Functional Interface  표준  API 패키지
- 크게 Consumer, Supplier, Function, Operator, Predicate로 구분된다. 

#### Consumer 
Consumer 함수적 인터페이스의 특징은 **리턴값이 없는** accept() 메소드를 가지고 있다. accpet()메소드는 단지 **매개값을 소비하는 역할만** 한다. (사용만 할뿐 리턴값이 없다는 의미이다.)

매개변수의 타입과 수에 따라서 아래와 같은 Consumer들이 있다.
![Alt text](./1550734256357.png)

Consumer<T> 인터페이스를 타겟 타입으로 하는 람다식은 다음과 같이 작성할 수 있다.
```java
Consumer<Stinrg>consumer = t -> { t 를 소비하는 실행문; }
```


#### Supplier 함수적 인터페이스
Supplier 함수적 인터페이스의 특징은 매개 변수가 없고 **리턴값이 있는 getXXX()** 메소드를 가지고 있다. 이 메소드들은 **실행 후 호출한 곳으로 데이터를 리턴**하는 역할을 한다.
![Alt text](./1550734659008.png)

Supplier<T>인터페이스를 다겟 타입으로 하는 람다식은 다음과 같이 작성할 수 있다.
getAsInt() 메소드가 매개값을 가지지 않으므로 람다식도 ()를 사용한다. 

```java
	IntSupplier suppler = () -> {...; return int_value; }
    public static void main(String[] args) {
        IntSupplier intSupplier = () -> {
            int num = (int) (Math.random() * 6) + 1;
 
            return num;
        };
        int num = intSupplier.getAsInt();
        System.out.println("눈의 수 : " + num);
```

#### Function 함수적 인터페이스
Function 함수적 인터페이스의 특징은 **매개값과 리턴값이 있는 applyXXX()** 메소드를 가지고 있다. 이 메소드들은 **매개값을 리턴값으로 매핑하는 역할**을 한다. 

![Alt text](./1550735706691.png)

Function<T, R> - R apply(T t) - 객체 T를  객체 R로 매핑: 

Function<T, R> 인터페이스를 타겟 타입으로 하는 람다식은 다음과 같이 작성할 수 있다. apply() 메소드는 매개값으로 T타입 객체 하나를 가지므로, 람다식도 하나의 매개변수를 사용한다. apply()의 리턴값이 R이므로 람다식의 리턴값도 R객체가 된다.

```java

	Function<Student, String> function = t -> { return t.getName(); }; 
	or
	Function<Student, String> function = t -> t.getName();
```

### 1. forEach() method in Iterable interface
기존에 Collection을 순회 하려면 Iterator를 만들었어야한다. 잘못 사용하면 ConcurrentModificationException을 발생 시킨다.

JAVA8은 java.lang.Iterable interface에 forEach() method를 포함시켰다.  
**forEach() method는 java.util.function.Consumer**를 아규먼트로 받는다. 이렇게 비즈니스 로직을 쪼개서 재사용 할 수 있게 해준다. 

```java

		//creating sample Collection
		List<Integer> myList = new ArrayList<Integer>();
		for(int i=0; i<10; i++) myList.add(i);
		
		//traversing using Iterator
		Iterator<Integer> it = myList.iterator();
		while(it.hasNext()){
			Integer i = it.next();
			System.out.println("Iterator Value::"+i);
		}
		
		//traversing through forEach method of Iterable with anonymous class
		myList.forEach(new Consumer<Integer>() {

			public void accept(Integer t) {
				System.out.println("forEach anonymous class Value::"+t);
			}

		});
		
		//traversing with Consumer interface implementation
		MyConsumer action = new MyConsumer();
		myList.forEach(action);
		
	}
```


#맨날 까먹는 Null 정리

### spot the differences between the @NotNull, @NotEmpty, and @NotBlank

hibernate Validator 
- use JUnit, AssertJ for unit test

#### @NotNull - null은 X, Empty는 허용("") 
- @NotNull **must be not null**. An empty value, however, is perfectly legal.
- @NotNull: a constrained **CharSequence**, **Collection**, **Map**, or **Array** is valid as long as it’s not null, but it can be empty

#### @NotEmpty - null X, Empty 또한 X
- check that the size/length of the supplied object is greater than zero.
- @NotEmpty: a constrained **CharSequence**, **Collection**, **Map**, or **Array** is valid as long as it’s not null and its size/length is greater than zero
- @NotEmpty must be not null and its size/length must be greater than zero.
- 길이또한 정해주고 싶다면, 다음처럼

```java
	@NotEmpty(message = "Name may not be empty")
	@Size (min = 2, max = 32, message = "Must be between 2 and 32")
	private String name;
```

#### @NotBlank 
- @NotBlank: a constrained **String** is valid as long as it’s not null and the trimmed length is greater than zero


# Generics in JAVA

**Generic was added in JAVA 5 to provide compile-time type checking and removing risk of ClassCastException!!**

#### 중요
1. Class Casting Exception을 줄이기 위해  자바 5부터 등장(**Compile시간에 type Checking** 효율을 위해..)
2. **comile시간**에 type checking을 하고, **필요한 경우 type casting을 완료한다.** 그 후 type casting 관련  **byte code를 지운다**. 그렇기에 **run-time시에 type-cast 오버헤드는 없다!!!**  

```java 
	List<String>list = new ArrayList<String>();
	list.add("abc"); 
	list.add(new Integer(5)); <- Compiler Error..
	
	for(String str : list) {
		//no type casting needed, avoids ClassCastException!!
	}	
```
리스트 생성시에 Type을 String으로 지정하였다. 그리하여 다른 타입의 object를 add할때 compile time error를 던진다. 이렇게  Type을 지정해줌으로서 ClassCastException을 피할 수 있다.

## JAVA Generic Class

GenericType Class를 사용하면 type-casting이 필요없고, runtime에서 ClassCastException을 없앨수 있다. 그러나 type을 지정해주지 않고, 객체를 생성하면 compiler는 warning을 던진다("GenericType is a raw type.) 

Reference에는 GenericType<T>은 parameterized 되어야 한다고 나와있다. type 을 지정해주지 않으면 type은 Object 가 된다.  

```java
	GenericType<String> type = new GenericType<>();
	type.set("sangmin"); //valid
	
	GenericType type1 = new GeneircType(); //raw type!!
	type1.set("sangmin"); //valid
	type1.set(12); //valid and autoboxing support
```

@SupressWarnings("rawtypes") annotation을 붙여서 compiler wargning을 supress시킬수 있다.

## JAVA Generic Interface
Comparable Interface는 Generics의 좋은 예이다. 

```JAVA
	public interface Comparable<T> {
		public int compareTo(T o); 
	}
```

JAVA type Parameter 
- The Most Commonly used type parameter names are:
	- E = Element(used extendsively by the Java Collections Framwork)
	- K = Key (Used in Map)
	- N = Number
	- T = Type
	- V = Value(Used In Map)
	- S,U,V etc. = 2nd, 3rd, 4th types

## JAVA Generics Method

가끔 Whole Class를 Parmeterized 할순 없다. 이때 우리는 generics method를 생성할 수 있다. 
```JAVA
	public class GenericsMethods {
		public stastis <T> boolean isEqual(GenericsType<T> g1, GenericsType<T> g2) {
		retunr g1.getT().equals(g2.get());
		}
	}
	public static void main(String[]args) {
		GenericsType<String> g1 = new GeneircsType<>();
		GenericsType<String> g2 = new GenericsType<>();
		g1.setT("sangmin");
		g2.setT("sangmin");
		boolean isEqual = GenericsMethods.isEqual(g1,g2);
		//This feature, known as type interface!!!!!
		//allows you to invoke a generic method as on ordinary method. without specifying a type between angle brackets
		//compiler will infer the type that is needed
}
```
이 메소드(isEqual)를 호출할 때 type을 지정해서 파라미터에 보낼 수 있고, 기존 normal method처럼 사용가능하다. 따라서 이를 type interface라고 지칭한다.

## JAVA Generics Bounded Type Parameters 
- to restrict the type of objects that can be used in the parameterized type method

예를들어 두가지 object 를 비교하는데, object가 compare가능한지를 make sure하게 하는 메소드를 지정하고 싶을때. bounded type parameter를 지정한다. 

```JAVA
	public static <T extends Comparable<T>> int compare(T t1, T t2){
		return t1.compareTo(t2);
	}
```

이때, not Comparable한 class를 사용한다면 compiler time시 error를 뱉을것이다.
- Java Generics supports multiple bounds also!
- <T extends A & B & C> : A can be interface or clss. 
- A 가 class면 B, C는 interface여야한다. 

## JAVA Generics and Inheritance
```JAVA
MyClass<String> myClass1 = new MyClass<String>();
MyClass<Object> myClass2 = new MyClass<Object>();
myClass2 = myClass1; //compliation error since MyClass<String> is not a MyClass<Object>

Object obj = new Object();
obj = myClass1; // myClass<T> parnet is Object

public static clss MyClass<T> {} 
```

MyClass<String> variable을 MyClass<Object>에 할당할 수 없다. 관계가 엮여있지 않기 때문.
in fact MyClass<T> parent is Object!!!!


## JAVA Generic Classes and Subtyping

**ArrayList<E>의 상속관계**
ArrayList<E> implements List<E>,
										List<E> extends Collection<E>
따라서 ArrayList<String>은 List<String>의 **SubType**이고. List<String>은 Collection<String>의 **SubType**이다!!!!

The Subtyping relationship is preserved as long as we don't change the type argument, below shows an example of **multiple type parametes.**

```JAVA
	interface MyList<E, T> extends List<E> {
	}
```
The Subtypes of List<String> can be MyList<String, Object>, MyList<String, Integer> and so on.

## JAVA Generics WidCards 

**Question Mark(?) is the wildcard in generics and represent an unknown type.** The wildcard can be used as the type of a parameter, filed, or local variale and somtimes as a return type.

	1. upper bounded wildcards 
	2. lower bounded wildcards
	3. wildcard capture

### JAVA Generics Uppder Bounded Wildcard

```JAVA
	public static double sum(List<Number> list) {
		double sum = 0;
		for(Number n : list) {
			sum += n.doubleValue();
		}
		return sum;
	}
```
위의 sum method는 List<Integer> listInteger. List<Double> listDouble을 인자로 받지 못한다. Type을 Number로 지정해줬기 때문. <Integer>와 <Double>은 related 되어있지 않다. 이럴때 generics wildcard upper bound를 사용할 수 있다.

Upper bound class or interface를 활용하여 sub class type의 파라미터를 넘길 수 있다.

```JAVA
	public static double sum(List <? extends Number> list) {
		double sum = 0;
		for(Number n : list) {
			sum += n.doubleValue();
		}
		return sum;
	}
```

### JAVA Generics Unbounded Wildcard
- 가끔은 모든 타입이 허용되는 generic method를 사용할 상황이 있다. 
```JAVA
	public static void PrintData(List<?> list) {
		for(Object obj : list) {
			System.out.print(obj + "::");
		}
	}
```
List<?> list 는 List<? extends Object> 와 같은 표현이다. 

## JAVA Generics Lower Bounded Wildcard
Integer 리스트에 Integer를 넣는다고 가정해보자. argument를 List<Integer>로 하면 문제없겠지만, List<Number> list or List<Object> list 로 묶여있으며 Integer를 들고있으면 사용할 수 없다. 이럴때 Lower boud를 활용할 수 있다.

We use Generics wildcard(?) with **super** keyword and lower boud class to achieve this.

```JAVA
	public static void addIntegers(List<? super Integer> list) {
		list.add(new Integer(50));
	}
```

### Subtyping using Generics Wildcard
```JAVA
	List<? extends Integer> intList = new ArrayList<>();
	List<? extends Number> numList = intList;
```

## JAVA Generics Type Erasure 

Generics in JAVA는 **Complie Time**에 type checking을 한다. 따라서 rum-time에는 필요 없다. 그렇기에 java Compiler는 type-erasure를 사용한다. 바이트 코드 안의 모든 generics type checking코드를 type-erasure 를 사용해 지우고, 필요할때 type-casting을 진행한다. 

**결론적으로 generic은 런타임 오버헤드를 갖지 않는다!! (compile 타임때 필요하면 캐스팅 완료)**
