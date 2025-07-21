# item 79. 과도한 동기화는 피하라

과도한 동기화는 성능을 떨어뜨리고, 교착상태에 빠뜨리고, 심지어 예측할 수 없는 동작을 낳기도 한다...

응답 불가와 안전 실패를 피하려면 동기화 메서드나 동기화 블록 안에서는 제어를 절대로 클라이언트에 양도하면 안된다.

예를 들어 동기화된 영역 안에서는 재정의할 수 있는 메서드는 호출하면 안되며, 클라이언트가 넘겨준 함수 객체를 호출해서도 안된다.

동기화된 영역을 포함한 클래스 관점에서 이런 메서드는 모두 바깥 세상에서 온 외계인이다.

이런 메서드가 하는 일에 따라 동기화된 영역은 예외를 일으키거나, 교착상태에 빠지거나, 데이터를 훼손할 수도 있다.

ex. 집합에 원소가 추가되면 클라이언트에게 알림을 보내는 클래스

```java
// Broken - invokes alien method from synchronized block!
public class ObservableSet<E> extends ForwardingSet<E> {
    public ObservableSet(Set<E> set) {
        super(set);
    }

    private final List<SetObserver<E>> observers = new ArrayList<>();

    public void addObserver(SetObserver<E> observer) {
        synchronized (observers) {
            observers.add(observer);
        }
    }

    public boolean removeObserver(SetObserver<E> observer) {
        synchronized (observers) {
            return observers.remove(observer);
        }
    }

    private void notifyElementAdded(E element) {
        synchronized (observers) {
            for (SetObserver<E> observer : observers)
                observer.added(this, element);
        }
    }

    @Override
    public boolean add(E element) {
        boolean added = super.add(element);
        if (added)
            notifyElementAdded(element);
        return added;
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        boolean result = false;
        for (E element : c)
            result |= add(element); // Calls notifyElementAdded
        return result;
    }
}
```

관찰자들은 addObserver와 removeObserver 메서드를 호출해 구독을 신청하거나 해지한다.

ObservableSet은 잘 동작할 것처럼 보인다..

```java
//0부터 99를 잘 출력한다.
public static void main(String[] args) {
    ObservableSet<Integer> set =
        new ObservableSet<>(new HashSet<>());

    set.addObserver((s, e) -> System.out.println(e));

    for (int i = 0; i < 100; i++)
        set.add(i);
}
```

하지만 아래 코드처럼 집합에 추가된 정수값을 출력하다가, 그 값이 23이면 자기 자신을 구독해지하는 관찰자를 추가해보자.

```java
set.addObserver(new SetObserver<>() {
    public void added(ObservableSet<Integer> s, Integer e) {
        System.out.println(e);
        if (e == 23)
            s.removeObserver(this);
    }
});
```

`ConcurrentModificationException`을 던진다.  
관찰자의 added 메서드 내 removeObserver->observers.remove 메서드가 호출된 시점이 notifyElementAdded가 관찰자들의 리스트를 순회하는 도중이기 때문이다.

notifyElementAdded 메서드에서 수행하는 순회는 동기화 블록 안에 있으므로 동시 수정이 일어나지 않도록 보장하지만, 정작 자신이 콜백을 거쳐 되돌아와 수정하는 것까지 막지는 못한다.

이번에는 구독해지를 위해 removeObserver를 직접 호출하지 않고 다른 쓰레드를 호출해보자...

```java
// Observer that uses a background thread needlessly
set.addObserver(new SetObserver<>() {
    public void added(ObservableSet<Integer> s, Integer e) {
        System.out.println(e);
        if (e == 23) {
            ExecutorService exec = Executors.newSingleThreadExecutor();
            try {
                exec.submit(() -> s.removeObserver(this)).get();
            } catch (ExecutionException | InterruptedException ex) {
                throw new AssertionError(ex);
            } finally {
                exec.shutdown();
            }
        }
    }
});
```

예외는 나지 않지만, 교착상태에 빠진다.

백그라운드 쓰레드가 s.removeObserver를 호출하면 관찰자를 잠그려 시도하지만, 메인 쓰레드가 이미 락을 쥐고 있기 때문에 락을 얻을 수 없다.

다행히 위 예시들에서는 외계인 메서드(added)가 호출될 때, 동기화 영역이 보호하는 자원(관찰자)은 일관된 상태였다. (addObserver()나 removeObserver()로 수정이 끝나 읽기만 하는 시점)

하지만!! 락이 걸린 중간 상태에서 외부 메서드가 수행되면, 해당 메서드가 그 상태를 건드릴 수도 있고, 데이터 불일치, 예외, 프로그램 오작동이 생길 수 있다.

예를 들어

```java
synchronized(observers) {
    observers.add(new SpecialObserver());
    // 지금 observers 리스트는 임시 상태일 수 있음 (예: capacity 증가 중, 반복 중 삽입 등)
    someObserver.notify(this);  // 👈 여기서 alien method 호출
}
```

notify가 호출되는 시점에 observers는 반복 중이거나 remove로 인해 iterator가 무효 상태가 될 수 있음

그리고 notifyElementAdded는 동기화된 observers를 순회하는데, observer.added에서 removeObserver를 호출하면, removeOberser는 다시 synchronized(observers) 블록으로 진입해야 한다.

자바의 락은 재진입 가능하므로 같은 쓰레드가 락을 다시 요청해도 락 획득이 가능하다.  
데드락은 발생하지 않지만, 위에서 말한 이유로 일관성이 변경되므로 락의 역할을 수행하지 못하는 상황이 발생하는 것이다.

다행히 외계인 메서드 호출을 동기화 블록 바깥으로 이동하면 문제가 해결된다.

```java
// Alien method moved outside of synchronized block - open calls
private void notifyElementAdded(E element) {
    List<SetObserver<E>> snapshot = null;

    synchronized(observers) {
        snapshot = new ArrayList<>(observers);
    }

    for (SetObserver<E> observer : snapshot)
        observer.added(this, element);
}
```

기존의 문제는 동기화 블록 내에서 외부 메서드(alien method)를 호출하고,
그 메서드가 동기화 대상(observers)을 수정하려 했기 때문에 발생했다.

이제는 동기화 블록에서 복사본(snapshot)을 만든 뒤,
락을 해제한 상태로 복사본을 순회하기 때문에
동기화 대상이 반복 중 수정되어도 예외나 데드락 없이 일관성을 유지할 수 있다.

또한 동기화 영역 안에서 외부 메서드를 호출하는 경우, 다른 쓰레드들은 그 외부 메서드가 실행되는 동안 기다려야 한다.

위 코드처럼 동기화 영역 바깥에서 호출되는 외부 메서드를 열린 호출(open call)이라고 한다.  
열린 호출은 실패 방지 효과 외에도 동시성 효율을 크게 개선해준다.

자바 동시성 컬렉션 라이브러리의 `CopyOnWriteArrayList`는 정확히 이런 목적으로 설계된 클래스이다.

내부를 변경하는 작업은 항상 복사본을 만들어 수행하도록 구현되어 순회할 때 락이 필요 없어 빠르다.

수정할 일은 드물고 순회만 빈번히 일어나는 관찰자 리스트 용도로는 알맞다.

```java
// Thread-safe observable set with CopyOnWriteArrayList
private final List<SetObserver<E>> observers = new CopyOnWriteArrayList<>();

public void addObserver(SetObserver<E> observer) {
    observers.add(observer);
}

public boolean removeObserver(SetObserver<E> observer) {
    return observers.remove(observer);
}

private void notifyElementAdded(E element) {
    for (SetObserver<E> observer : observers)
        observer.added(this, element);
}
```

add와 addAll은 그대로 두고, 명시적으로 동기화한 부분이 사라졌다.

---

동기화 영역에서는 가능한 한 일을 적게 하라

멀티코어가 일반화된 오늘날에는 과도한 동기화가 병렬 실행 기회를 줄이고, 코어 간 메모리 일관성을 맞추기 위한 지연을 초래해 성능을 떨어뜨릴 수 있다.

가변 클래스를 작성하려거든,

1. 동기화를 전혀 하지 말고 그 클래스를 동시에 사용해야 하는 클래스가 외부에서 알아서 동기화 하게 하자.

   ```java
   List<String> list = new ArrayList<>(); // 스레드 안전하지 않음

   synchronized (list) {
       list.add("hi"); // 외부에서 락 걸기
   }
   ```

2. 동기화를 내부에서 수행해 쓰레드 안전한 클래스로 만들자.  
   단, 클라이언트가 외부에서 객체 전체에 락을 거는 것보다 동시성을 월등히 개선할 수 있을 때만 유효

   ```java
   public class ThreadSafeCounter {
       private final AtomicInteger count = new AtomicInteger();

       public void increment() {
           count.incrementAndGet(); // 내부에서 스레드 안전 처리
       }
   }
   ```

   클래스를 내부에서 동기화하기로 했다면, 락 분할(lock splitting), 락 스트라이핑(lock striping), 비차단 동시성 제어(nonblocking concurrency control) 등 다양한 기법으로 동시성을 높여줄 수 있다.
