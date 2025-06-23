# item 30. 이왕이면 제네릭 메서드로 만들라

클래스와 마찬가지로 메서드도 제네릭으로 만들 수 있다.

방법은 제네릭 타입 작성법과 비슷하다

## ex. 두 집합의 합집합을 반환하는 메서드

```java
// Uses raw types - unacceptable! (Item 26)
public static Set union(Set s1, Set s2) {
    Set result = new HashSet(s1);
    result.addAll(s2);
    return result;
}
```

컴파일은 되지만 경고가 2개지요

```text
// 컴파일 시 경고
Union.java:5: warning: [unchecked] unchecked call to HashSet(Collection<? extends E>) as a member of raw type HashSet
    Set result = new HashSet(s1);
                          ^
Union.java:6: warning: [unchecked] unchecked call to addAll(Collection<? extends E>) as a member of raw type Set
    result.addAll(s2);
                 ^
```

s1, s2, result 다 raw type이다...

메서드 선언에서 s1, s2, result의 원소 타입을 타입 매개변수로 명시하고,  
메서드 안에서도 이 타입 매개변수만 사용하게 수정하자

```java
// Generic method
public static <E> Set<E> union(Set<E> s1, Set<E> s2) {
    Set<E> result = new HashSet<>(s1);
    result.addAll(s2);
    return result;
}
```

위 코드에서 타입 매개변수 목록은 \<E> 이고, 반환 타입은 Set\<E> 이다.

```java
// Simple program to exercise generic method
public static void main(String[] args) {
    Set<String> guys = Set.of("Tom", "Dick", "Harry");
    Set<String> stooges = Set.of("Larry", "Moe", "Curly");
    Set<String> aflCio = union(guys, stooges);
    System.out.println(aflCio);
}
```

```
[Moe, Tom, Harry, Larry, Curly, Dick]
```

위 union 메서드는 집합 3개의 타입이 모두 같아야 했지만,  
한정적 와일드카드 타입을 쓰면 더 유연하게 개선할 수 있다.

## 제네릭 싱글턴 팩토리

때때로 불변 객체를 여러 타입으로 활용할 수 있도록 만들어야 할 때가 있다.

예를 들어 항등함수(identity function, x -> x)을 담은 클래스를 만들고 싶다고 해보자.

```java
UnaryOperator<String> f1 = x -> x;
UnaryOperator<Integer> f2 = x -> x;

System.out.println(f1.apply("hello")); // "hello"
System.out.println(f2.apply(42));      // 42
```

f1, f2는 동일한 로직을 수행하지만, 타입만 다르다.

제네릭은 런타임에 타입 정보가 소거되므로 하나의 객체를 어떤 타입으로든 매개변수화 할 수 있다.

-> 제네릭을 이용해 해결??? 제네릭 객체는 결국 같은 Object 처럼 취급되니까

```java
UnaryOperator<Object> identity = x -> x;

// 제네릭 타입으로 캐스팅해도 실제로는 같은 객체
UnaryOperator<String> stringFn = (UnaryOperator<String>) identity;
UnaryOperator<Integer> intFn = (UnaryOperator<Integer>) identity;

System.out.println(stringFn.apply("abc")); // "abc"
System.out.println(intFn.apply(123));      // 123
```

캐스팅을 String, Integer로 했지만 둘은 결국 같은 객체다.  
그리고 클라이언트가 직접 캐스팅을 수행해야 하므로 타입 안정성이 떨어진다

-> 제네릭 싱글턴 팩토리 구현

```java
// 요청한 타입 매개 변수에 맞게 매번 그 객체의 타입을 바꿔줌
public class IdentityFunction {
    private static final UnaryOperator<Object> IDENTITY_FN = x -> x;

    @SuppressWarnings("unchecked")
    public static <T> UnaryOperator<T> identityFunction() {
        return (UnaryOperator<T>) IDENTITY_FN;
    }
}
```

```java
// 사용 예시
UnaryOperator<String> stringFn = IdentityFunction.identityFunction();
UnaryOperator<Double> doubleFn = IdentityFunction.identityFunction();

System.out.println(stringFn.apply("hi"));  // "hi"
System.out.println(doubleFn.apply(3.14));  // 3.14
```

## 재귀적 타입 한정(Recursive type bound)

자기 자신이 들어간 표현식으로 타입 매개변수를 제한할 수 있다.

### ex. 컬렉션에서 가장 큰 값을 구하는 메서드를 만들고 싶다...

```java
List<Integer> numbers = List.of(3, 7, 2);
Integer max = max(numbers); // 7
```

어떤 값이 더 큰지 비교해야 하는데 어떻게 할 수 있을까  
-> Comparable 인터페이스를 이용하면 된다

```java
public interface Comparable<T> {
    int compareTo(T o);
}
```

여기서 타입 매개변수 T는 비교할 수 있는 원소의 타입을 정의한다.  
어떤 클래스가 Comparable<T>를 구현하면, 그 클래스는 T 타입의 객체와 비교할 수 있다는 뜻

```java
public class Person implements Comparable<Person> {
    public int age;

    @Override
    public int compareTo(Person other) {
        return Integer.compare(this.age, other.age);
    }
}
```

Person 객체는 다른 Person 객체와 비교 가능함

---

우리가 만들고 싶은 max 함수는 이런 형태인데,

```java
public static <E> E max(Collection<E> c)
```

E가 Comparable을 구현한 타입이어야 compareTo를 호출해서 비교를 할 수 있음

따라서 타입에 제한을 걸어야 함. (Bound)

```java
// Using a recursive type bound to express mutual comparability
public static <E extends Comparable<E>> E max(Collection<E> c);
```

타입 한정인  
`<E extends Comparable<E>>`는  
E는 반드시 자기 자신과 비교 가능한 타입이어야 한다 라는 의미임

Comparable\<E> 에 자기 자신 E가 다시 등장해서 재귀적 타입 한정 이라고 부름

```java
// Returns max value in a collection - uses recursive type bound
public static <E extends Comparable<E>> E max(Collection<E> c) {
    if (c.isEmpty())
        throw new IllegalArgumentException("Empty collection");

    E result = null;
    for (E e : c)
        if (result == null || e.compareTo(result) > 0)
            result = Objects.requireNonNull(e);

    return result;
}
```
