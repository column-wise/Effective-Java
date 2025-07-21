# item 78. 공유 중인 가변 데이터는 동기화해 사용하라

`synchronized` 키워드는 해당 메서드나 블록을 한번에 한 쓰레드씩 수행하도록 보장한다.

즉, 객체를 하나의 일관된 상태에서 다른 일관된 상태로 변화함을 보장한다.

\+ 동기화된 메서드나 블록에 들어간 쓰레드가 같은 락의 보호 아래 수행된 모든 이전 수정의 최종 결과를 보게 해준다.(가시성)

자바 언어 명세는 쓰레드가 필드를 읽을 때 항상 '수정이 완전히 반영된' 값을 얻는다고 보장하지만, 쓰레드가 저장한 값이 다른 쓰레드에게 '보이는가'는 보장하지 않는다.

따라서 동기화는 배타적 실행뿐 아니라 쓰레드 사이의 안정적인 통신에 꼭 필요하다.

ex. 잘못된 코드

```java
// Broken! - How long would you expect this program to run?
public class StopThread {
    private static boolean stopRequested;

    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested)
                i++;
        });

        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
    }
}

```

첫번째 쓰레드는 stopRequest라는 boolean 변수를 polling하고,  
두번째 쓰레드는 1초 후에 stopRequest를 true로 바꾼다.

이론적으로 1초 후에 종료되어야 할 것처럼 보이지만, 종료되지 않을 수 있다.

위에서 언급한 가시성 때문인데, 두번째 쓰레드에서 1초 후에 stopRequest를 true로 변경하더라도, 첫번째 쓰레드에서는 확인이 불가할 수 있기 때문이다.

```java
// Properly synchronized cooperative thread termination
public class StopThread {
    private static boolean stopRequested;

    private static synchronized void requestStop() {
        stopRequested = true;
    }

    private static synchronized boolean stopRequested() {
        return stopRequested;
    }

    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested())
                i++;
        });

        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        requestStop();
    }
}
```

stopRequest에 synchronized 키워드를 붙이면 쓰레드간 가시성이 보장되어 1초 후 프로그램이 종료된다.

동기화는 배타적 수행과 쓰레드 간 통신이라는 두 가지 기능을 수행하는데,  
위 코드는 통신 목적으로만 사용된 것이다.

volatile 키워드를 이용하면 배타적 수행과는 상관 없지만 항상 최근에 기록된 값을 읽어옴을 보장할 수 있게 되기 때문에 위 코드에서 synchronized를 volatile로 대체해도 정상 동작한다.

```java
// Cooperative thread termination with a volatile field
public class StopThread {
    private static volatile boolean stopRequested;

    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested)
                i++;
        });

        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
    }
}
```

하지만 volatile은 주의해서 사용해야 한다.

ex. 일련번호 생성 프로그램

```java
// Broken - requires synchronization!
private static volatile int nextSerialNumber = 0;

public static int generateSerialNumber() {
    return nextSerialNumber++;
}
```

매번 고유한 값을 반환할 의도였지만, 동기화 없이 volatile 키워드만 사용하면 올바로 동작하지 않는다.

증가 연산자(++)은 nextSerialNumber 필드에 두 번 접근(읽기, 쓰기)하게 되는데,  
만약 다른 쓰레드가 이 두 번의 접근 사이에 값을 읽어가면 증가 연산이 의미 없어진다.

volatile 대신 synchronized 키워드를 붙이면 여러 쓰레드가 동시에 이 메서드를 호출해도 서로 간섭하지 않으며 이전 호출이 변경한 값을 읽게 되어 문제가 해결된다.

아직 끝이 아니다!!!!!

`java.util.concurrent.atomic` 패키지의 `AtomicLong`을 사용해보자.  
volatile은 통신 기능만 지원하지만, 이 패키지는 원자성(배타적 실행)까지 지원한다.  
성능도 동기화 버전보다 우수하다.

```java
// Lock-free synchronization with java.util.concurrent.atomic
private static final AtomicLong nextSerialNum = new AtomicLong();

public static long generateSerialNumber() {
    return nextSerialNum.getAndIncrement();
}
```

이번 아이템에서 언급한 문제들을 피하는 가장 좋은 방법은 애초에 가변 데이터를 공유하지 않는 것이다....

불변 데이터만 공유하거나 가변 데이터는 단일 쓰레드에서만 쓰도록 하자....

## 정리

여러 쓰레드가 가변 데이터를 공유한다면, 그 데이터를 읽고 쓰는 동작은 반드시 동기화 해야 한다.

동기화하지 않으면 한 쓰레드가 수행한 변경을 다른 쓰레드가 보지 못할 수도 있다.

배타적 실행은 필요 없고 쓰레드 간 통신만 필요하다면 volatile 한정자만으로 동기화 할 수 있다.
