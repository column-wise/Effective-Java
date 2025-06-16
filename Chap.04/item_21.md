# item 21. 인터페이스는 구현하는 쪽을 생각해 설계하라

Java 8 이전

인터페이스에 새 메서드를 추가하면 기존 구현체에서 컴파일 에러 발생

인터페이스에 새로운 메서드가 추가될 일은 영원히 없다고 가정하고 클래스가 작성됨

Java 8

디폴트 메서드가 도입되어  
기존 구현체에도 컴파일 오류 없이 새 메서드가 주입됨

하지만 모든 상황에서 불변식을 해치지 않는 디폴트 메소드를 작성하기는 어려웠음

### `removeIf` 예제

```java
default boolean removeIf(Predicate<? super E> filter) {
    Objects.requireNonNull(filter);
    boolean result = false;
    for (Iterator<E> it = iterator(); it.hasNext(); ) {
        if (filter.test(it.next())) {
            it.remove();
            result = true;
        }
    }
    return result;
}
```

주어진 predicate가 true를 반환하는 모든 원소를 제거한다.

기존 Collection 구현체 중 `org.apache.commons.collections4.collection.SynchronizedCollection`이 새 default 메서드와 충돌

클라이언트가 제공한 객체로 락을 거는 기능이 있었는데, removeIf를 오버라이드 하지 않아 메서드 호출을 동기화하지 못하게 되었다.

removeIf 디폴트 메서드는 동기화에 대해 아무것도 모르므로 락 객체를 사용할 수가 없다.  
-> 지금은 수정됨

<br>

---

<br>  
기존 인터페이스에 디폴트 메서드로
