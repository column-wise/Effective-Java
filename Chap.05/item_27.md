# item 27. 비검사 경고를 제거하라

제네릭을 사용하기 시작하면 많은 컴파일러 경고를 보게 될 것이다....

비검사 형변환, 비검사 메서드 호출, 비검사 매개변수화 가변인수 타입, 비검사 변환 등등

하지만 대부분의 비검사 경고는 쉽게 제거할 수 있다.

ex.

```java
// new HashSet() 은 raw type 생성
Set<Lark> exaltation = new HashSet();
```

컴파일하면 아래같은 경고가 나옴:

```
Venery.java:4: warning: [unchecked] unchecked conversion
    Set<Lark> exaltation = new HashSet();
                            ^
        required: Set<Lark>
        found: HashSet
```

-> 해결법

```java
Set<Lark> exaltation = new HashSet<Lark>();
```

또는

```java
Set<Lark> exaltation = new HashSet<>();
```

자바 7부터 지원하는 다이아몬드 연산자로 타입 추론

---

### 할 수 있는 한 모든 비검사 경고를 제거하라

런타임에 `ClassCastException`이 발생할 일이 없다

### 경고를 제거할 수는 없지만 타입 안전하다고 확신한다면 `@SuppressWarnings("unchecked") 어노테이션을 달아 경고를 숨겨라

단!!!! 타입 안전함을 검증하지 않은채 숨기면 안되고  
항상 가능한 한 좁은 범위에 적용해야 한다.

ex. ArrayList의 toArray 메서드

```java
public <T> T[] toArray(T[] a) {
    if (a.length < size)
        return (T[]) Arrays.copyOf(elements, size, a.getClass());
    System.arraycopy(elements, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```

다음과 같은 경고가 발생

```
ArrayList.java:305: warning: [unchecked] unchecked cast
    return (T[]) Arrays.copyOf(elements, size, a.getClass());
                                ^

    required: T[]
    found: Object[]
```

elements는 ArrayList의 필드 Object[] 이기 때문

-> `@SuppressWarnings("unchecked")` 를 추가  
메서드 전체에 달면 필요 이상으로 범위가 넓어져 반환값을 담을 지역변수를 하나 선언하고 그 위에 달았다

```java
public <T> T[] toArray(T[] a) {
    if (a.length < size) {
        // This cast is correct because the array we're creating
        // is of the same type as the one passed in, which is T[].
        @SuppressWarnings("unchecked") T[] result =
            (T[]) Arrays.copyOf(elements, size, a.getClass());
        return result;
    }
    System.arraycopy(elements, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```

그리고 `@SuppressWarnings("unchecked")` 를 사용해야 하면 꼭 그 이유를 주석으로 남겨라
