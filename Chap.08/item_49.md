# item 49. 매개변수가 유효한지 검사하라

## 메서드 바디가 실행되기 전에 유효성 검사

메서드와 생성자 대부분은 입력 매개변수의 값이 유효할 것을 기대한다.

매개 변수는 메서드 바디가 실행되기 전에 검사해야 한다.  
그렇게 하면 잘못된 값이 넘어왔을 때 곧바로 예외를 던질 수 있다.

그렇게 하지 않으면 메서드가 수행되는 중간에 모호한 예외를 던지며 실패할 수도 있고,  
더 나쁜 경우 메서드는 문제 없이 수행됐지만, 어떤 객체를 이상한 상태로 만들어서 미래에 이 메서드와 관련 없는 오류가 발생하게 된다.

자바 7에 추가된 `java.util.Objects.requireNonNull` 메서드를 이용하면 null 검사를 수동으로 하지 않아도 된다.

```java
// Inline use of Java's null-checking facility
this.strategy = Objects.requireNonNull(strategy, "strategy");
```

strategy가 null이면 NullPointerException을 발생시키고, 아니면 그대로 할당한다.  
메시지 "strategy"는 예외 메시지로 사용됨.

자바 9에서는 Objects에 범위 검사 기능도 더해졌다.

checkFromIndexSize, checkFromToIndex, checkIndex 인데, 리스트와 배열 전용으로 설계되었고, 예외 메시지를 지정할 수 없고, 닫힌 범위(양 끝단 포함)는 다루지 못하지만, 유용하다.


<br>

## 매개변수의 제약을 문서화

public과 protected 메서드는 매개변수 값이 잘못됐을 때 던지는 예외를 문서화해야 한다.

보통 던지는 에러는 `IllegalAgumentException`, `IndexOutOfBoundsException`, `NullPointerException`

```java
/**
 * Returns a BigInteger whose value is (this mod m). This method
 * differs from the remainder method in that it always returns a 
 * non-negative BigInteger.
 *
 * @param m the modulus, which must be positive
 * @return this mod m
 * @throws ArithmeticException if m is less than or equal to 0
 */
public BigInteger mod(BigInteger m) {
    if (m.signum() <= 0)
        throw new ArithmeticException("Modulus <= 0: " + m);
    
    // ... Do the computation
}
```

`@throws` 어노테이션을 이용하여 매개변수 제약을 어겼을 때 던지는 에러를 명시했다.

<br>

## Assertion

공개되지 않은 메서드(private, package-private)에서는 assert(단언)을 이용하여 인자 체크를 해도 된다.

```java
// Private helper function for a recursive sort
private static void sort(long a[], int offset, int length) {
    assert a != null;
    assert offset >= 0 && offset <= a.length;
    assert length >= 0 && length <= a.length - offset;
    // ... Do the computation
}
```

단언문은 단언한 조건이 무조건 참이라고 선언하는 것이다.

비공개 함수는 개발자가 호출을 통제하기 때문에 잘못된 인자가 들어오지 않음을 보장할 수 있다.  
-> 따라서 assert를 사용해도 된다

assert는 기본적으로 꺼져 있기 때문에, 성능 부담이 없고 실행 시에도 아무런 영향이 없다.  
assert를 활성화하려면 JVM 실행 시 java -ea 또는 java -enableassertions 옵션을 주어야 한다.

개발할 때만 옵션을 키고, 운영, 배포 환경에서는 옵션을 끄면 됨

<br>

## 메서드가 직접 사용하지 않으나 나중에 쓰기 위해 저장하는 매개변수

특히 더 신경 써서 검사해야 한다.

생성자가 받는 매개변수도 이 케이스에 해당되는데, 검사를 생략하게 되면 다른 메서드에서 오류를 내기 때문에 이 값을 어디서 가져왔는지 추적하기 어려워 디버깅이 힘들어진다.

