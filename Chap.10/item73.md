# item 73. 추상화 수준에 맞는 예외를 던지라

## 예외 번역(exception translation)
메서드가 저수준 예외를 처리하지 않고 바깥으로 전파하는 일이 종종 일어나는데,  
이것은 개발자를 당황스럽게 할 뿐만 아니라 내부 구현 방식을 드러내 윗 레벨 API를 오염시킨다.

이 문제를 피하려면 상위 계층에서 저수준 예외를 잡아 자신의 추상화 수준에 맞는 예외로 바꿔 던지는 처리(예외 번역)를 해주어야 한다.

```java
// Exception Translation
try {
    ... // Use lower-level abstraction to do our bidding
} catch (LowerLevelException e) {
    throw new HigherLevelException(...);
}
```

다음은 `AbstractSequentialList`에서 수행하는 예외 번역의 예이다.
```java
/**
* Returns the element at the specified position in this list. 
* @throws IndexOutOfBoundsException if the index is out of range 
* ({@code index < 0 || index >= size()}).
*/
   public E get(int index) {
       ListIterator<E> i = listIterator(index);
       try {
           return i.next();
       } catch (NoSuchElementException e) {
           throw new IndexOutOfBoundsException("Index: " + index);
       }
}
```

## 예외 연쇄(exception chaining)
예외를 번역할 때, 저수준 예외가 디버깅에 도움이 된다면 문제의 근본 원인인 저수준 예외를 고수준 예외에 실어 보낼 수 있다.

이것을 예외 연쇄라고 하며, 별도의 접근자 메서드(`Throwable.getCause`)를 통해 저수준 예외를 꺼내 볼 수 있다.

```java
// Exception Chaining
try {
    ... // Use lower-level abstraction to do our bidding
} catch (LowerLevelException cause) { 
    throw new HigherLevelException(cause);
}
```

대부분의 표준 예외는 예외 연쇄용 생성자를 가지고 있지만, 없는 예외라면 `Throwable.initCause` 메서드를 이용해 `getCause`로 접근 가능한 원인을 주입해줄 수 있다.

## 정리
무턱대로 예외를 전파하는 것보다 예외 번역이 우수한 방법이지만,  
그렇다고 남용해서는 안된다.

가능하다면 저수준 메서드가 반드시 성공하도록 하여 아래 예층에서는 예외가 발생하지 않도록 하는 것이 최선이다.

아래 계층에서의 예외를 피할 수 없다면, 상위 계층에서 그 예외를 조용히 처리하여 문제를 API 호출자까지 전파하지 않는 방법도 좋다.  
이 경우 발생한 예외는 로깅 기능을 활용하여 기록해두면 사용자에게 문제를 전파하지 않고도 개발자가 로그를 분석해 추가 조치를 취할 수 있게 해준다.