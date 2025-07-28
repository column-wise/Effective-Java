# item 84. 프로그램의 동작을 쓰레드 스케줄러에 기대지 말라

여러 쓰레드가 실행 중이면 운영체제의 쓰레드 스케줄러가 어떤 쓰레드를 얼마나 오래 실행할 지 정한다.

구체적인 스케줄링 정책은 운영체제마다 다를 수 있다.  
따라서 잘 작성된 프로그램이라면 이 정책에 좌지우지돼서는 안된다.

정확성이나 성능이 쓰레드 스케줄러에 따라 달라지는 프로그램이라면 다른 플랫폼에 이식하기 어렵다.

견고하고 빠릿하고 이식성 좋은 프로그램을 작성하는 가장 좋은 방법은  
실행 가능한 쓰레드의 평균적인 수를 프로세서 수보다 지나치게 많아지지 않도록 하는 것이다.

## Busy Waiting 피하기

실행 가능한 쓰레드 수를 적게 유지하는 주요 기법은  
각 쓰레드가 무언가 유용한 작업을 완료한 후에는 다음 일거리가 생길 때까지 대기하도록 하는 것이다...

쓰레드는 당장 처리해야 할 작업이 없다면 실행돼서는 안된다.  
절대 바쁜 대기(Busy Waiting) 상태가 되면 안된다는 뜻이다.

`CountDownLatch`를 삐딱하게 구현한 코드를 살펴보자.

```java
// Awful CountDownLatch implementation - busy-waits incessantly!
public class SlowCountDownLatch {
    private int count;

    public SlowCountDownLatch(int count) {
        if (count < 0)
            throw new IllegalArgumentException(count + " < 0");
        this.count = count;
    }

    public void await() {
        while (true) {
            synchronized(this) {
                if (count == 0)
                    return;
            }
            // Busy-waiting: consumes CPU without yielding
        }
    }

    public synchronized void countDown() {
        if (count != 0)
            count--;
    }
}
```

count가 0이 될때까지 루프를 돌며 계속 체크

저자의 PC에서 자바의 `CoundDownLatch` 와 속도를 비교하니 약 10배가 느렸다고한다.

이런 시스템은 성능과 이식성이 떨어질 수 있다.

## Thread.yield 피하기

특정 쓰레드가 다른 쓰레드들과 비교해 CPU 시간을 충분히 얻지 못해서 간신히 돌아가는 프로그램을 보더라도 `Thread.yield`를 써서 문제를 고쳐보려는 유혹을 떨쳐내자...

증상이 어느 정도는 호전될 수도 있지만, 이식성은 떨어진다.

처음 JVM에서는 성능을 높여준 yield가 두번째 JVM에서는 효과 없고, 세 번째 JVM에서는 오히려 느리게 동작하도록 만들 수 있다.

Thread.yield는 테스트할 수단도 없다.

차라리 애플리케이션 구조를 바꿔 동시에 실행 가능한 쓰레드 수가 적어지도록 조치하자.

쓰레드 우선순위는 자바에서 이식성이 가장 나쁜 특성에 속한다.  
쓰레드 몇 개의 우선순위를 조율해서 애플리케이션의 반응 속도를 높이는 방법이 유효할 수도 있겠지만, 그래야 할 상황은 드물고 이식성이 떨어진다.

## 정리

- 프로그램의 동작을 쓰레드 스케줄러에 기대지 말자. 견고성과 이식성 ↓
- Thread.yield와 쓰레드 우선순위에 의존해서도 안된다.
- 쓰레드 우선순위는 이미 잘 동작하는 프로그램의 서비스 품질을 높이기 위해 드물게 쓰일 수는 있지만, 간신히 동작한느 프로그램을 '고치는 용도'로 사용해서는 절대 안된다.
