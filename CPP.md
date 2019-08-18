C++ TIPS
==========

### 임시 객체와 이동 시맨틱 

```cpp
class String {
public:
    String() {
        std::cout << "constructor " << std::endl;
        strData = nullptr;
        len = 0;
    };

    String(const char *str) {
        std::cout << " String(const char*) constructor " << std::endl;
        len = strlen(str);
        alloc(len);
        strcpy(strData, str); // 깊은 복사
    }

    String(const String &rhs) {
        std::cout << " String(const char*) constructor " << std::endl;
        alloc(len);
        strcpy(strData, rhs.strData);
        len = rhs.len;
    }

    String &operator=(const String &rhs) {
        if (this != &rhs) {
            release();
            alloc(len);
            strcpy(strData, rhs.strData);
            len = rhs.len;
        }
        return *this;
    }

    ~String() {
        std::cout << " ~String() deconstrcutor " << std::endl;
        release();
        strData = nullptr;
    }

    char *GetStrData() const {
        return strData;
    }

    int GetLen() const {
        return len;
    }

private:
    void alloc(int len) {
        strData = new char[len + 1];
        std::cout << "strData allocated : " << (void*) strData << std::endl;
    }

    void release() {
        delete[] strData;
        if (strData)
            std::cout << "strData released : " << (void*) strData << std::endl;
    }
    char *strData;
    int len;
};

```

위와 같은 String class가 있고, 이 클래스는 복사 생성자와 연산자 대입 생성자가 존재한다. 
그리고 다음과 같은 String getName() 메소드와 메인 함수가 존재한다.

```cpp
String getName() {
    String res("TEST");
    return res;
}

int main() {
    String a;
    a = getName();
}
```

main의 a = getName()은 a에 getName의 반환 객체를 복사 생성자를 통해 초기화 시킨다. 
이때 정상적인 플로우라면 깊은복사가 발생하지 않아야 한다. (간단하게 깊은복사는 각각 메모리 할당)

그러나 getName()이 반환하는 res를 보자. 이는 메소드 내의 객체로서 getName() 메소드가 res를 반환하는 순간 소멸된다. 따라서 a에게 getName()의 반환값을 전달해주는 무언가가 필요하고 이를 임시객체라고 한다. getName()은 res를 임시객체에 복사하게되고 자신은 소멸되는 형태이다. 

임시객체는 이름이 없고, 복사가 완료 받은 후 a에 복사된다. 이후 임시 객체는 사라진다. 정상적인 플로우라면 깊은복사는 일어나지 않아야 정상이나, res -> 임시객체, 임시객체 -> a 라는 2번의 깊은복사가 일어나게 된다. 

getName() 메소드 반환값 리턴시 사용되는 복사 생성자를 변화시켜 임시객체로 얕은 복사를 실행하고, a = getName()시 진행했던 깊은 복사 또한 얕은 복사로 바꾸어야한다. 이 과정을 거치면 getName() 안의 String res("TEST")는 메모리 할당 과정을 거친 객체가 생성되고, 이를 가지고 얕은복사를 통해 임시객체를 생성한다. 마지막으로 복사 대입 연산자 대신 얕은복사를 통해 a에 값을 전달한다. 

즉 깊은 복사를 하지 않아도 된다는 얘기이다. 얕은 복사를 하기 위해 c++에서는 **이동 시맨틱을** 제공한다. move.. (쉽게 설명하면 strData를 가리키는 객체를 변화시키는 것!, res -> 임시객체 -> a)

**이동 시맨틱을 구현하기 위해선 R-VALUE**가 필요함. 뭔지는 알아서 찾아보길 바람. 아주 간단히 L-VALUE는 메모리 할당 상태인 변수이자 수정 가능, R-VALUE는 수정 안됨.
결론적으로 임시객체는 메모리 할당 상태이긴 하지만, 수정은 불가능함 따라서 R-VALUE 

#### 임시객체는 R-VALUE다. 

String &&r - getName(); // getName이 리턴하는 임시객체

**&&referenct는 R-VALUE 참조자**

즉 getName이 리턴하는 임시객체를 참조하는 변수를 선언하고, 이를 매개변수로 가지는 생성자를 만들면 됨.



