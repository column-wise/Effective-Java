# item 26. 로 타입은 사용하지 말라

클래스와 인터페이스 선언에 타입 매개변수(type-parameter)가 쓰이면 이를 **제네릭 클래스** 혹은 **제네릭 인터페이스**라고 한다.

ex. List\<E>

제네릭 클래스와 제네릭 인터페이스를 통틀어 제네릭 타입(generic type)이라고 한다.  
각각의 제네릭 타입은 일련의 매개변수화 타입(parameterized type)을 정의한다.

ex. List\<String>은 원소의 타입이 String인 리스트를 뜻하는 매개변수화 타입이고,  
`String`이 정규(formal) 타입 매개변수 `E`에 해당하는 실제(actual) 타입 매개변수이다.

## Raw Type

제네릭 타입을 하나 정의하면 그에 딸린 로 타입(raw type)도 함께 정의된다.

ex. List\<E>의 로 타입은 List

### 예시

```java
// Collection의 raw type
private final Collection stamps = ... ;
```

위 코드를 사용하면 실수로 도장 대신 동전을 넣어도 오류 없이 실행된다.

```java
// Iterator의 raw type
for(Iterator i = stamps.iterator(); i.hasNext(); ) {
    Stamp stamp = (Stamp) i.next(); // <- ClassCastException 발생
    stamp.cancel();
}
```

로 타입은 사용하지 말자  
로 타입을 사용하면 제네릭을 사용함으로써 얻을 수 있는 안전성과 표현력을 잃게 된다.

## Parameterized Type

제네릭을 활용하면 좋다

```java
private final Collection<Stamp> stamps = ... ;
```

이렇게 하면 컴파일러는 stamps에 Stamp의 인스턴스만 넣어야 함을 인지할수 있다.

```
// Stamps에 Coin 인스턴스를 넣으면 에러가 나는 모씁
Test.java:9: error: incompatible types: Coin cannot be converted
to Stamp
    c.add(new Coin());
                ^
```

---

List 같은 raw type은 사용하면 안되지만, List\<Object> 처럼 임의 객체를 허용하는 parameterized type은 괜찮다.  
=> 오잉????

```java
List list = new ArrayList();
list.add("hello");
list.add(123); // 문제 없이 컴파일됨

List<Object> objList = new ArrayList<Object>();
objList.add("hello"); // 가능
objList.add(123);     // 가능

List<String> strList = new ArrayList<String>();
objList = strList; // ❌ 컴파일 에러: List<String>은 List<Object>가 아님
```

아무거나 넣을 수 있는 것은 동일하지만  
List는 제네릭 타입에서 완전히 발을 뺀것이고  
List<Object>는 모든 타입을 허용한다는 의도를 컴파일러에게 명시한 것

\+ 매개변수로 List를 받는 메서드에는 List\<String>을 넘길 수 있지만 List\<Object> 를 받는 메서드에는 List\<String>을 넘길 수 없다

-> Generic은 공변(Covariant)하지 않음

```java
public static void main(String[] args) {
    List<String> strings = new ArrayList<>();
    unsafeAdd(strings, Integer.valueOf(42));
    String s = strings.get(0);
}

private static void unsafeAdd(List list, Object o) {
    list.add(0);
}
```

unsafeAdd는 List 로 타입을 받는 메서드로, list 자리에 아무 List\<E>나 다 넣을 수 있음

```
Test.java:10: warning: [unchecked] unchecked call to add(E) as a
member of the raw type List
    list.add(o);
            ^
```

경고는 나지만 컴파일은 가능

여기서 이제 unsafeAdd에서 List 대신 List\<Object>로 바꾸면

```
Test.java:5: error: incompatible types: List<String> cannot be
converted to List<Object>
    unsafeAdd(strings, Integer.valueOf(42));
        ^
```

컴파일조차 되지 않음

오류는 런타임보다는 컴파일에 찾는게 무조건 좋기 때문에 개이득임

## Wild Card

원소의 타입을 모를 때 로 타입을 쓰고 싶어질 수도 있다...

ex.
두 개의 집합을 받아서 공통 원소 개수를 반환하는 메서드를 만들고 싶다

```java
// 잘못된 방법(raw type 사용)
static int numElementsInCommon(Set s1, Set s2) {
    int result = 0;
    for(Object o1: s1) {
        if(s2.contains(01)) {
            result ++;
        }
    }

    return result;
}
```

작동은 하지만 로 타입을 사용해서 안전하지 않다....  
그럼 어떡하면 좋나

와일드카드(Wild Card)를 사용해라

```java
static int numElementsInCommon(Set<?> s1, Set<?> s2) {...}
```

?? 뭔 차이냐

-> 이러면 s1, s2에 null을 제외한 어떤 원소도 넣을 수 없게 되서 타입 불변식을 유지할 수 있다. (읽기 전용이 됨)

```
WildCard.java:13: error: incompatible types: String cannot be
converted to CAP#1
    c.add("verboten");
        ^
    where CAP#1 is a fresh type-variable:
        CAP#1 extends Object from capture of ?
```

다만 <?> 가 너무 제한적이라 불편하다면

- 제네릭 메서드: `<T> void method(Set<T> s)`
- bounded wildcard: `Set<? extends Number>` 또는 `Set<? super Integer>`

---

그리고 예외적으로 raw type을 써야 하는 경우도 있는데,

1. 클래스 리터럴(class literal): `SomeType.class` 클래스를 나타내는 객체를 얻을 때 사용

```java
List.class       // ✅ 가능 (raw type 사용)
String[].class   // ✅ 배열 가능
int.class        // ✅ 기본형 가능
List<String>.class  // ❌ 컴파일 에러
List<?>.class       // ❌ 마찬가지로 에러
```

2. instanceof 에서는 raw type 만 허용

```java
if(obj instanceof Set) {    // raw type
    Set<?> s = (Set<?>) obj;
     // Set<?> 로의 캐스팅은 타입 안전성이 보장되어 경고가 없음
}
```

## 정리

로 타입을 사용하면 런타임에 예외가 일어날 수 있으니 사용하면 안된다.  
로 타입은 제네릭이 도입되기 이전 코드와의 호환성을 위해 제공될 뿐
