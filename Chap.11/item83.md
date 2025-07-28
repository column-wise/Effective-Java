# item 83. 지연 초기화는 신중히 사용하라

지연 초기화(Lazy Initialization)는 필드의 초기화 시점을 그 값이 처음 필요할 때까지 늦추는 것이다.

그래서 그 값이 전혀 쓰이지 않으면 초기화도 일어나지 않는다.

다른 모든 최적화와 마찬가지로 지연 초기화에 대해 해줄 최선의 조언은  
"필요할 때까지는 적용하지 말아라" 이다...

지연 초기화는 양날의 검이기 때문인데  
클래스 혹은 인스턴스 생성 시의 초기화 비용은 줄지만 그 대신 지연 초기화하는 필드에 접근하는 비용이 커진다.

그럼에도 지연 초기화가 필요할 때가 있다.  
해당 클래스의 인스턴스 중 그 필드를 사용하는 인스턴스의 비율이 낮은 반면, 그 필드를 초기화하는 비용이 크다면 지연 초기화가 제 역할을 해줄 것이다.

하지만 정말 그런지 확인하려면 지연 초기화 적용 전후의 성능을 측정해보아야 한다.

멀티쓰레드 환경에서는 지연 초기화가 특히 까다롭다.  
지연 초기화하는 필드를 둘 이상의 쓰레드가 공유한다면 어떤 형태로든 반드시 동기화해야 한다.

대부분의 상황에서 일반적인 초기화가 지연 초기화보다 낫다.

일반적인 필드 초기화

```java
// Normal initialization of an instance field
private final FieldType field = computeFieldValue();
```

아래는 지연 초기화를 적용한 것이다.  
`synchronized`를 붙여 여러 쓰레드가 field 값을 동시에 읽어 computeFieldValue가 여러 번 수행되지 않도록 했다.

```java
// Lazy initialization of instance field - synchronized accessor
private FieldType field;

private synchronized FieldType getField() {
    if (field == null)
        field = computeFieldValue();
    return field;
}
```

성능 때문에 정적 필드를 지연 초기화해야 한다면 지연 초기화 홀더 클래스(Lazy Initialization Holder Class) 관용구를 사용하자.

```java
private static class FieldHolder {
    static final FieldType field = computeFieldValue();
}

private static FieldType getField() { return FieldHolder.field; }
```

getField가 처음 호출되는 순간 FieldHolder.field가 처음 읽히면서, 비로소 FieldHolder 클래스 초기화를 촉발한다.

그 이후에는 `field`는 이미 초기화된 상수 값으로 남아있고, `getField()`는 단순히 그 값을 반환할 뿐 `computeFieldValue()`는 다시 수행되지 않는다.

`getField` 메서드가 필드에 접근하면서 동기화를 전혀 하지 않으니 성능이 느려질 거리가 전혀 없다.  
이것이 이 패턴의 핵심 장점이다.

일반적인 VM은 오직 클래스를 초기화할 때만 필드 접근을 동기화할 것이다.  
클래스 초기화가 끝난 후에는 VM이 동기화 코드를 제거하여 그 다음부터는 아무런 검사나 동기화 없이 필드에 접근하게 된다.

---

성능 때문에 인스턴스 필드를 지연 초기화 해야 한다면 이중 검사(Double-Check) 관용구를 사용해라.

이 관용구는 초기화된 필드에 접근할 때의 동기화 비용을 없애준다.

필드의 값을 두 번 검사하는데, 한 번은 동기화 없이 검사하고, 필드가 아직 초기화되지 않았다면 두 번째는 동기화하여 검사한다.

두 번째 검사에서도 필드가 초기화되지 않았을 때만 필드를 초기화한다.  
필드가 초기화된 후로는 동기화하지 않으므로 해당 필드는 반드시 `volatile`로 선언해야 한다.

```java
// Double-check idiom for lazy initialization of instance fields
private volatile FieldType field;

private FieldType getField() {
    FieldType result = field;
    if (result == null) { // First check (no locking)
        synchronized(this) {
            if (field == null) // Second check (with locking)
                field = result = computeFieldValue();
        }
    }
    return result;
}
```

이중검사를 정적 필드에도 적용할 수 있지만, 굳이 그럴 필요는 없다.  
지연 초기화 홀더 클래스 방식이 더 낫다.

이중검사에는 변종이 있는데, 반복해서 초기화해도 상관없는 인스턴스 필드를 지연 초기화 해야 할 때가 있다.  
이 경우에는 두 번째 검사를 생략할 수 있다.

단일검사(Single-check) 관용구가 된다.

```java
// Single-check idiom - can cause repeated initialization!
private volatile FieldType field;

private FieldType getField() {
    FieldType result = field;
    if (result == null)
        field = result = computeFieldValue();
    return result;
}
```

모든 쓰레드가 필드의 값을 다시 계산해도 상관없고, 필드의 타입이 long과 double을 제외한 다른 기본 타입이라면, 단일검사의 필드 선언에서 volatile 한정자를 없애도 된다.

이 변종은 짜릿한 단일검사(Racy Single-Check) 관용구라 불린다.

이 관용구는 어떤 환경에서는 필드 접근 속도를 높여주지만, 초기화가 쓰레드당 최대 한 번 더 이뤄질 수 있다.

거의 쓰지는 않는다.
