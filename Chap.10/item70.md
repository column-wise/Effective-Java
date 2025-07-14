# item 70. 복구할 수 있는 상황에는 검사 예외를, 프로그래밍 오류에는 런타임 예외를 사용하라.

## 자바의 예외 계층 구조

![Java Exception Hierarchy](https://www.javamex.com/tutorials/exceptions/ExceptionHierarchy.png)

- 검사 예외(Checked Exception)
- 런타임 예외(Runtime Exception)
- 에러(Error)

## 언제 어떤 예외를 사용해야 하는가??

1. 호출하는 쪽에서 복구하리라 여겨지는 상황이라면 검사 예외를 사용하라.

   검사 예외를 던지면 호출자가 그 예외를 catch로 잡아 처리하거나 더 바깥으로 전파하도록 강제하게된다.

   메서드 선언에 포함된 예외 각각은 그 메서드를 호출했을 때 발생할 수 있는 유력한 결과임을 API 사용자에게 알려주는 것이다.

2. 프로그래밍 오류를 나타낼 때는 런타임 예외를 사용하자.

   런타임 예외의 대부분은 전제조건을 만족하지 못했을 때 발생한다.  
   예를 들어 배열의 인덱스는 0에서 (배열크기 - 1) 사이여야 한다.  
   ArrayIndexOutOfBoundsException이 발생했다는 건 이 전제조건이 지켜지지 않았다는 것이다.

## 애매하긴 해

위 지침에서 문제가 있다면, 복구할 수 있는 상황인지, 프로그래밍 오류인지 항상 명확하게 구분되지 않는다는 점이다..

예를 들어 자원 고갈은 말도 안되는 크기의 배열을 할당해 생긴 프로그래밍 오류일 수도 있고, 진짜로 자원이 부족해서 발생한 문제일 수도 있다.

자원이 일시적으로만 부족하거나 수요가 순간적으로만 몰린 것이라면 충분히 복구할 수 있는 상황이다.

## 사용자 정의 Error, Throwable은 피하라

에러는 보통 JVM이 자원 부족, 불변식 깨짐 등 더 이상 수행을 계속할 수 없는 상황을 나타낼 때 사용한다.

따라서 Error 클래스를 상속해 하위 클래스를 만드는 일은 자제하기 바란다.

다시 말해 여러분이 구현하는 비검사 throwable은 모두 RuntimeException이 하위 클래스여야 한다.

Exception, RuntimeException, Error를 상속하지 않는 throwable을 만들 수도 있겠지만,  
이것은 이로울게 없으니 절대로 사용하지 말자 !

throwable은 정상적인 검사 예외보다 나을게 없으면서 API 사용자를 헷갈리게 할 뿐이다.

## 예외도 결국 객체다

예외 클래스는 필드와 메서드를 가질 수 있다.  
즉, 단순히 메세지만 전달하는게 아니라 추가 정보를 담는 필드를 만들고, 그걸 제공하는 메서드를 만들어야 한다.

```java
public class FileReadException extends IOException {
    private final String filePath;

    public FileReadException(String filePath, Throwable cause) {
        super("Could not read file: " + filePath, cause);
        this.filePath = filePath;
    }

    public String getFilePath() {
        return filePath;
    }
}
```

이처럼 getFilePath() 같은 메서드를 제공하면 예외를 잡은 쪽에서 추가 정보를 안전하게 활용할 수 있다.

## 예외 메세지를 파싱하지 마라

```java
catch (FileReadException e) {
    if (e.getMessage().contains("myfile.txt")) {
        // ...
    }
}
```

이런 식으로 예외 메세지를 파싱해서 필요한 정보를 추출하는 것은 매우 나쁜 방법이다.  
메세지 포맷은 구현마다, 버전 마다 달라질 수 있어 비이식성(nonportable), 취약성(fragile) 높음

필드 + 메서드로 명시적으로 제공하라
