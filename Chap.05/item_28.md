# item 28. 배열보다는 리스트를 사용하라

배열과 제네릭 타입에는 아주 중요한 차이가 있다.

## 1. 배열은 공변(Covariant), 제네릭은 불공변(Invariant)

Sub가 Super의 하위 타입이라면, 배열 `Sub[]`는 배열 `Super[]`의 하위 타입이 된다.

반면 `List<Sub>` 와 `List<Super>`는 아무런 관계가 없어버림

ex.

```java
Object[] objectArray = new Long[1];
objectArray[0] = "타입이 달라 넣을 수 없다" ;
```

위 코드는 컴파일이 가능, 런타임에서 `ArrayStoreException`을 던진다.  
=> 문제 있다.

```java
List<Object> ol = new ArrayList<Long>();
ol.add("타입이 달라 넣을 수 없다");
```

이 코드는 컴파일이 되지 않는다.

## 2. 배열은 실체화(Reify)된다.

배열은 런타임에도 자신이 담기로 한 원소의 타입을 인지하고 확인한다.

그래서 위 예시 코드에서도 보듯 Long 배열에 String을 넣을 수가 없는 것이다.

반면 제네릭은 타입 정보를 컴파일 단계에서만 확인하고, 런타임에는 소거(erasure)된다.  
이것은 제네릭이 지원되기 전 레거시 코드와 제네릭이 호환되도록 하는 매커니즘이다.

## 그래서 배열과 제네릭은 잘 호환되지 않는다.

배열은 제네릭 타입, 매개변수화 타입, 타입 매개변수로 사용할 수 없다.

제네릭 타입의 배열을 생성할 수 없다.  
제네릭의 타입 정보는 지워지는 반면 배열은 타입 정보를 런타임에도 유지하기 때문

### ex. 컴파일 에러나는 케이스

- `new List<E>[]`
- `new List<String>[]`
- `new E[]`

### ex. 만약에 제네릭 배열 생성이 가능하다면?

```java
// Why generic array creation is illegal - won't compile!
List<String>[] stringLists = new List<String>[1];   // (1)
List<Integer> intList = List.of(42);                // (2)
Object[] objects = stringLists;                     // (3)
objects[0] = intList;                               // (4)
String s = stringLists[0].get(0);                   // (5)
```

제네릭 배열을 생성하는 (1)이 허용되어 컴파일이 된다면,  
(2)는 원소가 하나인 List\<Integer>를 생성한다.  
(3)은 List\<Integer>의 배열을 Object 배열에 할당한다. 배열은 공변하니까 가능하다(List가 Object의 subset이니까)  
(4)는 Object 배열의 첫 원소 자리에 (2)에서 만든 List\<Integer>의 인스턴스를 넣는다.  
제네릭은 타입이 소거되니까 List\<Integer>가 List가 되고, List\<Integer>[] 인스턴스의 타입은 List[]가 된다.

stringLists는 List<String>만 담겠다고 했는데 List<Integer>가 저장되어 버렸다.  
(5)에서는 이 배열의 첫 리스트에서 첫 원소를 꺼내려고 하는데,  
List<String> 으로 선언됐으므로 컴파일러는 원소를 String으로 형변환하려고 하고,  
여기서 `ClassCastException`이 발생

이런 상황을 방지하기 위해 제네릭 배열 생성을 막는 것이다.

---

E, List\<E>, List\<String> 같은 타입을 실체화 불가 타입(Non-reifiable type) 이라고 하는데,  
런타임에는 컴파일타임보다 타입 정보를 적게 가진다는 뜻이다.

소거 매커니즘 때문에 매개변수화 타입 중에 실체화 될 수 있는 타입은 List<?> 와 Map<?,?> 같은 비한정적 와일드카드 타입 뿐이다.

ex.

1. 실체화 불가

   ```java
   // non-reifiable 예시, 타입이 소거되기 때문
   List<String> list = new ArrayList<>();

   if (list instanceof List<String>) { // ❌ 컴파일 에러
       System.out.println("List of String");
   }
   ```

2. 비한정적 와일드카드는 실체화 가능

   ```java
   List<?> wildcardList = new ArrayList<String>();

   if (wildcardList instanceof List<?>) { // ✅ 가능
       System.out.println("It's a list of something!");
   }

   ```

   아래 처럼 raw type 또는 unbounded wildcard로만 검사 가능

   ```java
   if (list instanceof List)         // ✅ 가능 (raw type)
   if (list instanceof List<?>)      // ✅ 가능 (unbounded wildcard)
   ```

---

#### 제네릭 배열 생성 금지 규칙은 불편하다

이런 것도 안된다  
제네릭 컬렉션에서 자신의 원소 타입을 담은 배열 반환

```java
public class MyList<E> {
    public E[] toArray() {
        return new E[10]; // ❌ 컴파일 에러!
    }
}
```

그리고 가변 인자 메서드랑 제네릭을 같이써도 경고가 발생한다

```java
@SafeVarargs
public static <T> void printAll(T... args) {
    for (T t : args)
        System.out.println(t);
}
```

T... args는 결국 T[] args 인데, T[] 가 실체화 불가 타입이라 "unchecked" 경고가 발생하는 것

### 배열 대신 제네릭 리스트를 쓰자

ex. 생성자에서 받은 컬렉션 중 랜덤한 값을 반환하는 `choose` 메서드가 있는 `Chooser` 클래스

```java
// Chooser - a class badly in need of generics!
public class Chooser {
    private final Object[] choiceArray;

    public Chooser(Collection choices) {
        choiceArray = choices.toArray();
    }

    public Object choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceArray[rnd.nextInt(choiceArray.length)];
    }
}
```

`choose`의 반환값이 Object라 매번 형 변환을 해서 사용해야 한다. 너무 불편함  
-> 제네릭을 도입하고 싶다....

이렇게 하면 어떨까

```java
// A first cut at making Chooser generic - won't compile
public class Chooser<T> {
    private final T[] choiceArray;

    public Chooser(Collection<T> choices) {
        choiceArray = choices.toArray(); // ❌ 컴파일 에러 발생
    }

    // choose method unchanged
}
```

컴파일하면 아래와 같은 에러가 뜬다

```
Chooser.java:9: error: incompatible types: Object[] cannot be
converted to T[]
        choiceArray = choices.toArray();
                                    ^
    where T is a type-variable:
    T extends Object declared in class Chooser
```

Collections.toArray()가 Object[]를 반환하므로 타입이 맞지 않는다

그럼 (T[]) choices.toArray(); 로만 고쳐주면 되나???

```
Chooser.java:9: warning: [unchecked] unchecked cast
        choiceArray = (T[]) choices.toArray();
                                            ^

    required: T[], found: Object[]
    where T is a type-variable:
T extends Object declared in class Chooser
```

이번에는 경고가 뜬다...

T가 무슨 타입인지 모르니까 컴파일러가 안전한지 보장할 수 없다는 뜻이다

경고도 없애려면 배열 대신 리스트를 쓰자

```java
// List-based Chooser - typesafe
public class Chooser<T> {
    private final List<T> choiceList;

    public Chooser(Collection<T> choices) {
        choiceList = new ArrayList<>(choices);
    }

    public T choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceList.get(rnd.nextInt(choiceList.size()));
    }
}
```
