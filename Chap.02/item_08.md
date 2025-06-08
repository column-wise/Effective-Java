# item 08. finalizer와 cleaner 사용을 피하라

`finalizer`와 `cleaner`는 객체가 GC 대상이 될 때 자동으로 정리 작업을 수행하도록 도와주는 매커니즘

## finalizer

자바 객체가 GC 직전에 호출되는 메서드를 정의  
`Object` 클래스의 `protected void finalize()` 메서드를 오버라이드 하면 객체가 GC 되기 직전에 이 메서드가 자동으로 호출됨.

```java
@Override
protected void finalize() throws Throwable {
    try {
        System.out.println("객체가 소멸되기 직전입니다!");
    } finally {
        super.finalize();
    }
}
```

요즘은 잘 사용되지 않는데, 그 이유는

- 불확실: 언제 호출될지 알 수 없음(GC 타이밍에 의존)
- 성능 문제: Finalizer 큐를 따로 관리하기 때문에 GC 속도를 떨어뜨림
- 보안 문제: 악의적인 오브젝트가 finalizer 에서 자기 자신을 다시 참조해 부활할 수 있음(resurrection)
- 리소스 누수: finalizer 에만 리소스 해제를 의존하면, 해제가 너무 늦거나 안 될 수 있음.

## cleaner

자바 9부터 도입된 `java.lang.ref.Cleaner`는 finalizer 보다 안전하고 명시적인 리소스 정리 방식  
외부 리소스(DB, 파일, 네이티브 자원 등)를 관리하는 객체가 GC 대상이 되었을 때, 클린업 작업을 수행할 수 있도록 등록하는 구조

```java
Cleaner cleaner = Cleaner.create();

class MyResource implements AutoCloseable {
    private static class State implements Runnable {
        @Override
        public void run() {
            System.out.println("리소스 정리됨");
        }
    }

    private final State state;
    private final Cleaner.Cleanable cleanable;

    MyResource() {
        this.state = new State();
        this.cleanable = cleaner.register(this, state); // 객체가 GC 되면 state.run() 호출됨
    }

    @Override
    public void close() {
        cleanable.clean(); // 명시적 정리도 가능
    }
}
```

`AutoCloseable`로 명시적 해제를 보장할 수 없는 환경(ex. 개발자가 `close()`를 깜빡할 수 있는 상황)  
리소스를 이중으로 관리하려고 할 때: 명시적 해제 + 한 번 더 해제 Cleaner  
같은 상황에서 사용 가능함

## 쓰지 마라

1. 정시 실행 불가
   finalizer와 cleaner는 GC 타이밍에 의존하므로 언제 실행될지 보장할 수 없음

   특히 finalizer thread는 낮은 우선순위로 실행되기 때문에,

   메모리가 급하게 필요한 시점에 청소 작업이 지연될 수 있음

   → 자원 해제 타이밍이 너무 늦거나 무한히 지연될 수 있음

2. 상태 변경에 사용 금지
   파일 삭제, 락 해제, DB 연결 종료 등 영구적인 상태 변화가 필요한 작업을
   finalizer나 cleaner에 맡기면 작업 보장이 불가능

   → 반드시 명시적인 close() 또는 try-with-resources로 처리해야 함

3. 예외 무시
   finalize() 또는 Cleaner의 Runnable.run() 안에서 발생한 예외는 조용히 무시됨

   로그도 남기지 않으며, 해당 객체의 정리 작업이 부분적으로 실패할 수 있음

   → 디버깅도 어렵고, 시스템 상태가 silently corrupt 될 수 있음

4. 심각한 성능 저하
   finalizer가 붙은 객체는 GC가 즉시 회수하지 못하고 finalizer 큐로 따로 넘겨야 하며,

   Finalizer Thread에서 finalize()가 호출되기 전까지 해당 객체는 살아 있는 것으로 간주됨

   → 메모리 회수 지연 + GC 부하 증가 + Throughput 저하

5. 보안 취약점 (finalizer 공격)
   finalize() 안에서 객체가 자기 자신을 정적 필드에 할당하면 GC되지 않음 (부활)

   이를 통해 생성 중 예외로 초기화가 끝나지 않은 객체가 시스템에 노출될 수 있음

   직렬화/역직렬화 과정에서도 마찬가지 위험 발생

   ```java
       // 공격 예시
       @Override
       protected void finalize() throws Throwable {
           Attacker.leaked = this;  // 아직 초기화도 안 끝난 객체가 살아남음
       }
   ```

   해결 방법: 아무 것도 하지 않는 finalize() 메서드를 명시적으로 작성하고 final로 선언하여 오버라이드 방지

   ```java

   @Override
   protected final void finalize() { }
   ```

## AutoCloseable을 사용하자

`AutoClosable`을 구현하고, 클라이언트에서 인스턴스를 다 쓰고 나면 close 메서드를 호출  
\+ 꿀팁: 각 인스턴스가 자신이 닫혔는지를 추적하는 필드를 놓으면 좋다.

```java
public class MyResource implements AutoCloseable {
    private boolean closed = false;

    public void doSomething() {
        if (closed) {
            throw new IllegalStateException("리소스가 이미 닫혔습니다.");
        }
        System.out.println("작업 수행 중...");
        // 리소스를 사용하는 실제 로직
    }

    @Override
    public void close() {
        if (!closed) {
            closed = true;
            System.out.println("리소스를 정리합니다.");
            // 리소스 해제 로직
        }
    }
}
```

사용 예시(try-with-resources)

```java
public class ResourceTest {
    public static void main(String[] args) {
        try (MyResource resource = new MyResource()) {
            resource.doSomething();
            resource.doSomething();  // OK
        }

        // try 블록 밖에서 사용하려 하면 예외 발생
        // resource.doSomething();  // 컴파일 오류 or IllegalStateException
    }
}
```

### finalizer와 cleaner는 도대체 어디에 쓰는거냐?

1. cleaner는 안전망 목적으로 사용 가능하다
   자원의 소유자가 close 메서드를 호출하지 않는 것에 대비

2. 네이티브 피어(native peer)와 연결된 객체에서 사용  
   네이티브 피어란 일반 자바 객체가 네이티브 메서드를 통해 기능을 위임한 네이티브 객체를 말한다......??  
   네이티브 피어는 자바 객체가 아니므로 GC가 그 존재를 알지 못한다.  
   cleaner와 finalizer에서 이 네이티브 피어 메모리 해제 처리를 해줄 수가 있다.  
   단 성능 저하를 감당할 수 없거나 네이티브 피어가 사용하는 자원을 즉시 회수해야 한다면 close 메서드를 사용해야 한다.

### cleaner를 안전망으로 사용하는 방법

Room 자원을 수거하기 전에 반드시 청소(clean)해야 한다고 가정

```java
// An autocloseable class using a cleaner as a safety net
public class Room implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();

    // Resource that requires cleaning. Must not refer to Room!
    // static 이어야만 함.. inner class는 바깥 객체를 참조하게 되기 때문에 참조가 생겨 Room이 GC가 안됨
    private static class State implements Runnable {
        int numJunkPiles; // Number of junk piles in this room

        State(int numJunkPiles) {
            this.numJunkPiles = numJunkPiles;
        }

        // Invoked by close method or cleaner
        @Override
        public void run() {
            System.out.println("Cleaning room");
            numJunkPiles = 0;
        }
    }

    // The state of this room, shared with our cleanable
    private final State state;

    // Our cleanable. Cleans the room when it’s eligible for gc
    private final Cleaner.Cleanable cleanable;

    public Room(int numJunkPiles) {
        state = new State(numJunkPiles);
        cleanable = cleaner.register(this, state);
    }

    @Override
    public void close() {
        cleanable.clean();
    }
}
```

#### 정리를 잘 하는 클래스

```java
public class Adult {
    public static void main(String[] args) {
        try (Room myRoom = new Room(7)) {
            System.out.println("Goodbye");
        }
    }
}
```

try-with-resource를 이용했기 때문에 close()가 보장되어 Cleaner는 필요하지 않다.

---

#### 방 청소를 하지 않는 건방진 녀석

```java
public class Teenager {
    public static void main(String[] args) {
        new Room(99);
        System.out.println("Peace out");
    }
}
```

close()가 호출되지 않으므로 Cleaner가 청소를 해줄까?????
-> 그것은 알 수 없음  
JVM은 `System.exit()`을 만나면 Cleaner를 반드시 실행하지는 않음......

<br>

### 결론

finalizer 쓰지 마라  
cleaner는 안전망 역할이나 중요하지 않은 네이티브 자원 회수용으로만 사용하자.  
물론 불확실성과 성능 저하를 감수해야 한다.
