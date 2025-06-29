# item 32. 제네릭과 가변인수를 함께 쓸 때는 신중하라

가변 인수(varargs)는 메서드에 넘기는 인수의 개수를 클라이언트가 조절할 수 있게 해주는데,  
가변인수 메서드를 호출하면 가변인수를 담기 위한 배열이 자동으로 런타임에 하나 만들어진다.

반면 제네릭은 런타임에 타입 정보가 소거된다.

그래서 실체화 불가 타입으로 varargs 매개 변수를 선언하면 컴파일러가 경고를 보낸다.

```
warning: [unchecked] Possible heap pollution from
    parameterized vararg type List<String>
```

매개변수화 타입의 변수가 타입이 다른 객체를 참조하면 힙 오염이 발생.  
이렇게 다른 타입 객체를 참조하는 상황에서는 컴파일러가 자동 생성한 형변환이 실패할 수 있어 제네릭이 제공하는 타입 안정성이 깨진다.

```java
// Mixing generics and varargs can violate type safety!
static void dangerous(List<String>... stringLists) {
    List<Integer> intList = List.of(42);
    Object[] objects = stringLists;
    objects[0] = intList; // Heap pollution
    String s = stringLists[0].get(0); // ClassCastException
}
```

제네릭 varargs 배열 매개변수에 값을 저장하는 것은 안전하지 않다.

제네릭 배열을 개발자가 직접 생성하는 것은 허용하지 않으면서 제네릭 varargs 배개변수를 받는 메서드를 선언할 수 있게 한 이유는 무엇일까

그것은 제네릭이나 매개변수화 타입의 varargs 매개변수를 받는 메서드가 실무에서 매우 유용하기 때문이다.
(Arrays.asList(T... a), Collections.addAll(Collections<? super T> c, T... elements) 등)

제네릭 varargs 매개변수를 사용하는 메서드가 안전하다는 걸을 보장할 수 있다면
`@SafeVarargs` 어노테이션으로 경고를 숨길 수 있다.

그렇다면 어떻게 안전하다고 확신할 수 있을까?

varargs 로 만들어진 제네릭 배열에 메서드가 아무것도 저장하지 않고(수정하지 않고)  
그 배열의 참조가 밖으로 노출되지 않는다면 타입 안전하다.

이때, varargs 매개변수 배열에 아무것도 저장하지 않고도 타입 안전성을 깰 수 있으니 조심해야 한다(?)

```java
// UNSAFE - Exposes a reference to its generic parameter array!
static <T> T[] toArray(T... args) {
    return args;
}
```

인자들을 받아 배열로 만들고 그대로 리턴하는 메서드  
이 배열의 실제 타입은 컴파일 타임에 추론된 타입에 따라 결정됨.

그래서 컴파일러는 잘못된 타입의 배열을 생성할 수도 있음

ex.

```java
static <T> T[] pickTwo(T a, T b, T c) {
    switch(ThreadLocalRandom.current().nextInt(3)) {
        case 0: return toArray(a, b);
        case 1: return toArray(a, c);
        case 2: return toArray(b, c);
    }
    throw new AssertionError(); // Can't get here
}
```

```java
public static void main(String[] args) {
    String[] attributes = pickTwo("Good", "Fast", "Cheap");
}
```

컴파일러는 toArray에 넘길 T 인스턴스 2개를 담을 varargs 매개변수 배열을 만드는 코드를 생성한다. (toArray(T ...))

이 코드가 만드는 배열의 타입은 Object[] 인데, pickTow에 어떤 타입의 객체를 넘기더라도 담을 수 있는 가장 구체적인 타입이기 때문이다.

pickTow의 반환값을 attributes에 저장하기 위해 (String[])을 컴파일러가 자동 생성하고, 여기서 `ClassCastException`이 발생한다.

다음은 제네릭 varargs 매개변수를 안전하게 사용하는 메서드다.

```java
// Safe method with a generic varargs parameter
@SafeVarargs
static <T> List<T> flatten(List<? extends T>... lists) {
    List<T> result = new ArrayList<>();
    for (List<? extends T> list : lists)
        result.addAll(list);
    return result;
}
```

이 메서드는 vararg 배열을 외부에 노출하지 않고, 읽기만 하기 때문에 안전하다

`@SafeVarargs` 어노테이션으로 경고를 가리는 것보다는  
[item 28](https://github.com/column-wise/Effective-Java/blob/main/Chap.05/item_28.md)의 조언에 따라 varargs 매개변수를 List 매개변수로 바꿀 수도 있다.

```java
// List as a typesafe alternative to a generic varargs parameter
static <T> List<T> flatten(List<List<? extends T>> lists) {
    List<T> result = new ArrayList<>();
    for (List<? extends T> list : lists)
        result.addAll(list);
    return result;
}
```

-> 타입 안전

정적 팩토리 메서드인 List.of를 활용하면 다음 코드처럼 이 메서드에 임의 개수의 인수를 넘길 수도 있다.

```java
audience = flatten(List.of(friends, romans, countrymen));
```

List.of 에도 `@SafeVarargs` 가 달려있기 때문....

또는 이렇게도 쓸 수 있다.

```java
static <T> List<T> pickTwo(T a, T b, T c) {
    switch(rnd.nextInt(3)) {
        case 0: return List.of(a, b);
        case 1: return List.of(a, c);
        case 2: return List.of(b, c);
    }
    throw new AssertionError();
}
```

```java
public static void main(String[] args) {
    List<String> attributes = pickTwo("Good", "Fast", "Cheap");
}
```

이 코드는 배열 없이 제네릭만 사용하므로 타입 안전하다.
