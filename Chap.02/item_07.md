# item 07. 다 쓴 객체 참조를 해제하라.

JVM에는 GC가 있지만, 메모리 관리에 더 이상 신경 쓰지 않아도 된다는 뜻은 아니다.

# 메모리 누수가 발생할 수 있는 대표적인 상황

## 1. 클래스가 직접 메모리(객체 참조)를 관리하는 경우우

```java
// Can you spot the "memory leak"?
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }
    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }
    public Object pop() {
        if (size == 0) throw new EmptyStackException();
        return elements[--size];
    }
    /**
    * Ensure space for at least one more element, roughly
    * doubling the capacity each time the array needs to grow.
    */
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

`pop`에서 문제가 발생.  
elements[size]에 저장된 객체의 참조는 남기 때문에

```java
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null; // ✨ 명시적 null 처리로 메모리 누수 방지
    return result;
}
```

이렇게 명시적으로 `null` 처리를 하게 되면, pop 된 인스턴스는 GC의 대상이 된다.

다만, 모든 객체를 사용후에 `null` 처리해 줄 필요는 없고, [참조를 담은 변수의 scope를 가능한 좁은 범위로 설정하면 자연스럽게 처리된다.](/Chap.08/item_57.md)

```java
{
    SomeObject o = new SomeObject();
    o.doSomething();
} // 여기서 o는 스코프를 벗어나므로, GC 가능
```

<br>

위 예시의 `Stack` 클래스가 메모리 누수에 취약한 이유는  
스택이 자신의 메모리를 직접 관리하기 때문이다.

GC는 스택 내의 어떤 원소들이 사용되고, 사용되지 않는지 알 수 없다.

따라서 `자기 메모리를 직접 관리하는 클래스라면 프로그래머는 항시 메모리 누수에 주의해야 한다.`

## 2. 캐시

캐시에 넣은 참조는 명시적으로 제거하지 않으면 GC가 회수하지 못함.  
키나 값이 오래 참조되면 메모리를 계속 차지하게 됨.

### 해결책

- 키에만 외부 참조가 존재할 경우: `WeakHashMap` 사용(키에 대한 참조가 없어지면 엔트리도 제거됨)
- 유효기간이 명확하지 않은 경우: `ScheduledThreadPollExecutor` 같은 백그라운드 쓰레드를 사용하거나 `LinkedHashMap.removeEldestEntry` 메서드를사용하여 정리한다.

## 3. 콜백 및 리스너

이벤트 기반 시스템에서 클라이언트가 등록만하고 해제하지 않으면, 콜백이 계속 참조되어 GC가 불가능함.

### 해결책

- 명시적으로 `removeListener()` 같은 해제 메서드를 구현
- WeakHashMap` 같은 약한 참조로 저장
