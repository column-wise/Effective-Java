# item 81. wait와 notify보다는 동시성 유틸리티를 애용하라

오늘날의 Java에서는 wait와 notify를 사용해야 할 이유가 많이 줄었다.

자바 5에서 도입된 고수준의 동시성 유틸리티가 wait와 notify로 하드코딩해야 했던 전형적인 일들을 대신 처리해주기 때문이다.

wait와 notify는 올바르게 사용하기가 매우 까다롭기까지 하니 고수준 동시성 유틸리티를 사용하자.

`java.util.concurrent`의 고수준 유틸리티는 세 범주로 나눌 수 있다.  
실행자 프레임워크, 동시성 컬렉션(Concurrent Collection), 동기화 장치(Synchronizer)다.

실행자 프레임워크는 item 80에서 가볍게 살펴보았다.

## 동시성 컬렉션

동시성 컬렉션은 `List`, `Queue`, `Map` 같은 표준 컬렉션 인터페이스에 동시성을 가미해 구현한 고성능 컬렉션이다.  
높은 동시성을 달성하기 위해 동기화를 각자의 내부에서 수행한다.

따라서 동시성 컬렉션에서 동시성을 무력화하는 것은 불가능하며, 외부에서 락을 추가로 사용하면 오히려 속도가 느려진다.

동시성 컬렉션에서 동시성을 무력화하지 못하므로 여러 메서드를 원자적으로 묶어 호출하는 일 역시 불가능하다.  
그래서 여러 기본 동작을 하나의 원자적 동작으로 묶는 '상태 의존적 수정' 메서드들이 추가되었다.

예를 들어 `Map`의 `putIfAbsend(key, value)` 메서드는 주어진 키에 매핑된 값이 없을 때만 새 값을 집어넣고 null을 반환,  
기존에 값이 있었다면 그 값을 반환한다.

다음은 `ConcurrentHashMap`을 이용해 `String.intern`의 동작을 흉내 내어 구현한 메서드이다.

```java
// Concurrent canonicalizing map atop ConcurrentMap - not optimal
private static final ConcurrentMap<String, String> map =
    new ConcurrentHashMap<>();

public static String intern(String s) {
    String previousValue = map.putIfAbsent(s, s);
    return previousValue == null ? s : previousValue;
}
```

`ConcurrentHashMap`은 get 같은 검색 기능에 최적화되어 있어  
get을 먼저 호출하여 필요할 때만 `putIfAbsent`를 호출하면 더 빠르다.

```java
// Concurrent canonicalizing map atop ConcurrentMap - faster!
public static String intern(String s) {
    String result = map.get(s);
    if (result == null) {
        result = map.putIfAbsent(s, s);
        if (result == null)
            result = s;
    }
    return result;
}
```

이 메서드는 작가의 PC에서 `String.intern` 보다 6배나 빨랐다.
`String.intern`에는 오래 실행되는 프로그램에서 메모리 누수를 방지하는 기술이 들어가 있다는 것을 감안해도 여전히 빠르다.

이처럼 동시성 컬렉션은 동기화한 컬렉션을 낡은 유산으로 만들어버렸다.

이제는 Collections.synchronizedMap 보다는 ConcurrentHashMap을 사용하는 것이 훨씬 좋다.

## 동기화 장치

동기화 장치는 쓰레드가 다른 쓰레드를 기다릴 수 있게 하여, 서로 작업을 조율할 수 있게 해준다.

자주 쓰이는 동기화 장치는 `CountDownLatch`와 `Semaphore`다.  
`CyclicBarrier`와 `Exchanger`는 그보다 덜 쓰인다.  
그리고 가장 강력한 동기화 장치는 `Phaser`다.

카운트다운 래치는 일회성 장벽으로, 하나 이상의 쓰레드가 또 다른 하나 이상의 쓰레드 작업이 끝날 때까지 기다리게 한다.

`CountdownLatch`의 생성자는 int 값을 받으며, 이 값이 래치의 `countDown` 메서드를 몇 번 호출해야 대기 중인 쓰레드들을 깨우는지를 결정한다.

이 간단한 장치를 활용하면 유용한 기능들을 아주 쉽게 구현할 수 있다.

예를 들어 어떤 동작들을 동시에 시작해 모두 완료하기까지의 시간을 재는 간단한 프레임워크를 구현한다고 해보자.

이 프레임워크는 메서드 하나로 구성되며, 메서드는 동작들을 실행할 실행자와 동작을 몇 개나 동시에 수행할 수 있는지를 뜻하는 동시성 수준(Concurrency)을 매개변수로 받는다.

타이머 쓰레드가 시계를 시작하기 전에 모든 작업자 쓰레드는 동작을 수행할 준비를 마친다.  
마지막 작업자 쓰레드가 준비를 마치면 타이머 쓰레드가 '시작 방아쇠'를 당겨 작업자 쓰레드들이 일을 시작하게 한다.

마지막 작업자 쓰레드가 동작을 마치자마자 타이머 쓰레드는 시계를 멈춘다.

이 기능을 wait, notify로 구현하려면 아주 난해하고 지저분한 코드가 되겠지만, `CountdownLatch`를 쓰면 놀랍도록 직관적으로 구현할 수 있다.

```java
// Simple framework for timing concurrent execution
public static long time(Executor executor, int concurrency,
                        Runnable action) throws InterruptedException {
    CountDownLatch ready = new CountDownLatch(concurrency);
    CountDownLatch start = new CountDownLatch(1);
    CountDownLatch done = new CountDownLatch(concurrency);

    for (int i = 0; i < concurrency; i++) {
        executor.execute(() -> {
            ready.countDown(); // Tell timer we're ready
            try {
                start.await(); // Wait till peers are ready
                action.run();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                done.countDown(); // Tell timer we're done
            }
        });
    }

    ready.await();              // Wait for all workers to be ready
    long startNanos = System.nanoTime();
    start.countDown();          // And they're off!
    done.await();               // Wait for all workers to finish
    return System.nanoTime() - startNanos;
}
```

ready 래치는 작업자 쓰레드들이 준비가 완료됐음을 타이머 쓰레드에 통지할 때 사용됨.

통지를 끝낸 작업자 쓰레드들은 두 번째 래치인 start가 열리기를 기다린다.  
마지막 작업자 쓰레드가 ready.countDown을 호출하면 타이머 쓰레드가 시작 시각을 기록하고 start.countDown을 호출하여 기다리던 작업자 쓰레드들을 깨운다.

타이머 쓰레드는 이제 세 번째 래치인 done이 열리기를 기다린다.  
done 래치는 마지막 남은 작업자 쓰레드가 동작을 마치고 done.countDown을 호출하면 열린다.

타이머 쓰레드는 done 래치가 열리자마자 깨어나 종료 시각을 기록한다.

※ time 메서드에 넘겨진 실행자 executor는 concurrent 매개변수로 지정한 동시성 수준만큼의 쓰레드를 생성할 수 있어야 한다.  
그렇지 못하면 메서드는 결코 끝나지 않을 것이다.  
이런 상태를 쓰레드 기아 교착상태(Thread Starvation Deadlock)이라고 한다.

`InterruptedException`을 캐치한 작업자 쓰레드는 `Thread.currentThread().interrupt()` 관용구를 사용해 인터럽트를 되살리고 자신은 run 메서드에서 빠져나온다.

이렇게 해야 실행자가 인터럽트를 적절하게 처리할 수 있다.

동시성 유틸리티를 살짝 알아봤는데, time 메서드에서 사용한 카운트 다운 래치 3개는 `CyclicBarrier` 혹은 `Phaser` 인스턴스 하나로 대체할 수도 있다.

더 간결해지겠지만 이해하기는 더 어려울 것이다.

### wait와 notify

어쩔 수 없이 레거시 코드에 있는 wait와 notify를 사용해야 할 일이 있을 수도 있다.

wait 메서드는 쓰레드가 어떤 조건이 충족되기를 기다리게 할 때 사용한다.

락 객체의 wait 메서드는 반드시 그 객체를 잠근 동기화 영역 안에서 호출해야 한다.

```java
// The standard idiom for using the wait method
synchronized (obj) {
    while (<조건이 충족되지 않았다>)
        obj.wait(); // (락을 놓고, 깨어나면 다시 잡는다.)

    ... // 조건이 충족됐을 때의 동작을 수행한다.
}
```

wait 메서드를 사용할 때는 반드시 대기 반복문(wait loop) 관용구를 사용하라.  
반복문 밖에서는 절대로 호출하지 말자.

이 반복문은 wait 호출 전후로 조건이 만족하는지를 검사하는 역할을 한다.

대기 전에 조건을 검사하여 조건이 이미 충족되었다면 wait를 건너뛰게 하는 것은 응답 불가 상태를 예방하는 조치다.

만약 조건이 이미 충족되었는데 쓰레드가 notify(notifyAll) 메서드를 먼저 호출한 후 대기 상태로 빠지면, 그 쓰레드를 다시 깨울 수 있다고 보장할 수 없다.

한편, 대기 후에 조건을 검사하여 조건이 충족되지 않았다면 다시 대기하게 하는 것은 안전 실패를 막는 조치다.

```java
synchronized (obj) {
    if (!condition) {
        obj.wait(); // 깨어났을 때 조건을 다시 안 봄
    }
    // 조건이 아직 false일 수도 있는데 실행됨 → ❗ 안전 실패 발생 가능
}
```

이렇게 loop가 아닌 if 문으로만 감싸져있으면 아래 경우에 대응할 수 없다

1. spurious wakeup(허위 깨어남)

   - wait()는 아무 이유 없이 깨어날 수 있음(JVM 스펙상 허용)
   - 따라서 wait()에서 깨어났다고 해서 조건이 자동으로 true라고 가정하면 안됨

2. 다른 쓰레드가 조건을 만족시키지 못하고 notify

   - 여러 쓰레드가 wait() 중일 수 있음
   - 다른 쓰레드가 notify()를 했지만 조건을 만족시킨 건 아닐 수도 있음

3. notify()와 깨어나는 사이에 다른 쓰레드가 상태를 변경하는 경우

   - 쓰레드 A가 조건을 만족시키고 notify()를 호출

   - 그런데 쓰레드 B가 먼저 깨어나 락을 획득하고 상태를 변경함

     → 쓰레드 A는 깨어났지만 조건이 더 이상 충족되지 않음

---

notify와 notifyAll 중 무엇을 선택하느냐 하는 문제도 있다.

일반적으로 notifyAll을 사용하는 것이 합리적이고 안전한 조언이 될 것이다.

다른 쓰레드까지 깨어날 수 있긴 하지만, while loop로 조건을 확인하여 충족되지 않았다면 다시 대기하도록 구현하면 문제될 것이 없다.

또한 관련 없는 쓰레드가 실수로 혹은 악의적으로 wait를 호출하여 notify를 삼켜버리는 경우에 대응할 수 있다.
