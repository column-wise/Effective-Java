# item 82. 쓰레드 안전성 수준을 문서화하라

한 메서드를 여러 쓰레드가 동시에 호출할 때 그 메서드가 어떻게 동작하느냐는 해당 클래스와 이를 사용하는 클라이언트 사이의 중요한 계약과 같다.

다음 목록은 쓰레드 안전성이 높은 순으로 나열한 것이다.

- 불변(Immutable)
- 무조건적 쓰레드 안전(Unconditionally Thread-Safe)
- 조건부 쓰레드 안전(Conditionally Thread-Safe)
- 쓰레드 안전하지 않음(not Thread-Safe)
- 쓰레드 적대적(Thread-Hostile)

이 분류는 『자바 병렬 프로그래밍』의 부록 A에 나오는 쓰레드 안전성 어노테이션(`@Immutable`, `@ThreadSafe`, `@NotThreadSafe`)과 대부분 일치한다.

무조건적 쓰레드 안전과 조건부 쓰레드 안전은 `@ThreadSafe` 어노테이션 밑에 속한다.

조건부 쓰레드 안전한 클래스는 주의해서 문서화해야 한다.

어떤 순서로 호출할 때 외부 동기화가 필요한지, 그리고 그 순서로 호출하려면 어떤 락 혹은 락들을 얻어야 하는지 알려줘야 한다.

클래스의 쓰레드 안전성은 보통 클래스의 문서화 주석에 기재하지만, 독특한 특성의 메서드라면 해당 메서드의 주석에 기재하도록 하자.  
열거 타입은 굳이 불변이라고 쓰지 않아도 된다.

클래스가 외부에서 사용할 수 있는 락을 제공하면 클라이언트에서 일련의 메서드 호출을 원자적으로 호출할 수 있다.

하지만 대가로 내부에서 처리하는 고성능 동시성 제어 메커니즘과 혼용이 어렵고  
클라이언트가 공개된 락을 오래 쥐고 놓지 않는 서비스 거부 공격(Denial-Of-Service Attack)을 수행할 수도 있게된다.

서비스 거부 공격을 막으려면 `synchronized` 메서드 대신 비공개 락 객체를 사용해야 한다.

```java
// Private lock object idiom - DoS 공격 차단
// 비공개 락 객체 관용구:
// 동기화에 this나 어떤 공개 객체를 사용하는 대신,
// 외부에 공개되지 않은 private final 객체를 락으로 사용하는 방식
private final Object lock = new Object();

public void foo() {
    synchronized(lock) {
        // 임계 구역
    }
}
```

외부에서는 `lock` 객체에 접근할 수 없으므로 클라이언트나 서브 클래스가 동기화 간섭을 할 수 없다.

※ `lock` 필드를 private으로 선언하지 않으면 심각한 동기화 오류가 발생할 수 있다.(item 78, 17 참고)

비공개 락 객체 관용구는 무조건적 쓰레드 안전 클래스에서만 사용할 수 있다.  
조건부 쓰레드 안전 클래스에서는 특정 호출 순서에 필요한 락이 무엇인지를 클라이언트에게 알려줘야 하므로 이 관용구를 사용할 수 없다.

그리고 비공개 락 객체 관용구는 상속용으로 설계한 클래스와 잘 맞는다.  
부모 클래스와 자식 클래스가 `this`를 락으로 공유하면 충돌이 발생할 수 있기 때문에 비공개 락 객체 관용구를 쓰는 것이 적절하다.

```java
// 부모 클래스
public class Base {
    public synchronized void baseMethod() {
        // this를 락으로 사용함
        System.out.println("Base method running");
    }
}
```

```java
// 자식 클래스
public class Derived extends Base {
    public synchronized void derivedMethod() {
        // 역시 this를 락으로 사용함
        System.out.println("Derived method running");
    }
}
```

`baseMethod()`와 `derivedMethod()`는 둘 다 this를 락으로 사용하는데, 같은 인스턴스를 기준으로 락을 잡으므로 하나가 실행 중이면 다른 하나는 진입하지 못한다.

비공개 락 객체를 사용하면?

```java
// 부모 클래스
public class Base {
    private final Object lock = new Object();

    public void baseMethod() {
        synchronized (lock) {
            System.out.println("Base method running");
        }
    }
}
```

```java
// 자식 클래스
public class Derived extends Base {
    private final Object childLock = new Object();

    public void derivedMethod() {
        synchronized (childLock) {
            System.out.println("Derived method running");
        }
    }
}
```

부모와 자식이 서로 다른 락 객체를 사용하므로 충돌하지 않는다.

## 정리

- 모든 클래스가 자신의 쓰레드 안전성 정보를 명확히 문서화 해야 한다.
  명확히 설명하거나 쓰레드 안전성 어노테이션을 사용할 수 있다.

- synchronized 한정자는 문서화와 관련이 없다.

- 조건부 쓰레드 안전 클래스는 메서드를 어떤 순서로 호출할 때 외부 동기화가 요구되고, 그 때 어떤 락을 얻어야 하는지도 알려줘야 한다.

- 무조건적 쓰레드 안전 클래스를 작성할 때는 `synchronized` 메서드가 아닌 비공개 락 객체를 사용하자.
