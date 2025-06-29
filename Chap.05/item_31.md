# item 31. 한정적 와일드카드를 사용해 API 유연성을 높여라

[item 28](https://github.com/column-wise/Effective-Java/blob/main/Chap.05/item_28.md)에서 이야기했듯 매개변수화 타입은 불공변(invariant)이다.

`List<String>`은 `List<Object>` 의 하위타입이 될 수 없다는 뜻이다.

하지만 때론 불공변 방식보다 유연한 무언가가 필요하다.

## ? extends E

item 29의 Stack을 보면

```java
public class Stack<E> {
    public Stack();
    public void push(E e);
    public E pop();
    public boolean isEmpty();
}
```

여기에 여러 원소들을 스택에 넣는 메서드를 추가해야 한다고 해보자.

```java
// pushAll method without wildcard type - deficient!
public void pushAll(Iterable<E> src) {
    for (E e : src)
    push(e);
}
```

컴파일에는 오류가 없지만 완벽하진 않다.  
`Stack<Number>`로 선언 후 pushAll(intVal)을 호출하면 어떻게 될까 (intVal : Integer)

Integer는 Number의 하위타입이니 잘 동작해야 할 것 같지만

```
StackTest.java:7: error: incompatible types: Iterable<Integer>
cannot be converted to Iterable<Number>
    numberStack.pushAll(integers);
                        ^
```

매개변수화 타입이 불공변이기 때문에 작동하지 않는다.

이것을 해결하기 위해 **한정적 와일드카드 타입**이라는 특별한 매개변수화 타입을 사용할 수 있다.

```java
// Wildcard type for a parameter that serves as an E producer
public void pushAll(Iterable<? extends E> src) {
    for (E e : src)
    push(e);
}
```

`Iterable<? extends E>` 는 'E 하위타입의 Iterable' 이라는 뜻이다.  
(extends라는 키워드가 알맞은 것 같지는 않지만)

## ? super E

이번에는 popAll 메서드를 만들어보자.

```java
// popAll method without wildcard type - deficient!
public void popAll(Collection<E> dst) {
    while (!isEmpty())
    dst.add(pop());
}
```

이번에도 주어진 컬렉션의 원소 타입이 스택의 원소 타입과 일치하면 문제 없겠지만  
`Stack<Number>` 의 원소를 Object용 컬렉션으로 옮긴다고 해보자.

```java
Stack<Number> numberStack = new Stack<Number>();
Collection<Object> objects = ... ;
numberStack.popAll(objects);
```

`Collection<Object>` 는 `Collection<Number>` 의 하위 타입이 아니라는  
pushAll 때와 비슷한 오류가 날 것이다.

```java
// Wildcard type for parameter that serves as an E consumer
public void popAll(Collection<? super E> dst) {
    while (!isEmpty())
    dst.add(pop());
}
```

이렇게 고치면 해결

**유연성을 극대화하려면 원소의 생산자나 소비자용 입력 매개변수에 와일드카드 타입을 사용하라.**

PECS: producer-extends, consumer-super

한편, 입력 매개변수가 생산자와 소비자 역할을 동시에 한다면 와일드카드 타입을 써도 좋을게 없다.

---

[item 30](https://github.com/column-wise/Effective-Java/blob/main/Chap.05/item_30.md) 의 union 함수도 보면

```java
public static <E> Set<E> union(Set<E> s1, Set<E> s2)
```

s1과 s2 모두 E의 생산자이니 PECS 공식에 따라

```java
public static <E> Set<E> union(Set<? extends E> s1, Set<? extends E> s2)
```

이게마따

```java
// 정상적으로 컴파일 됨 (자바 8부터)
Set<Integer> integers = Set.of(1, 3, 5);
Set<Double> doubles = Set.of(2.0, 4.0, 6.0);
Set<Number> numbers = union(integers, doubles);
```

### ※ 여기서 말하는 생산자, 소비자란?

함수형 프로그래밍에서의

- 생산자: () -> T
- 소비자: T -> ()

와는 다르다

여기서의 "생산자"란 나에게서 값을 꺼낼(read) 수 있다는 의미

```java
List<? extends Number> list = List.of(1, 2.5); // 읽기는 가능
Number n = list.get(0); // ✅ 생산됨 (read)
list.add(3); // ❌ 컴파일 오류 – 타입이 불확실해서 넣을 수 없음
```

```
Main.java:7: error: no suitable method found for add(int)
        list.add(3);
            ^
    method Collection.add(capture#1 of ? extends Number) is not applicable
      (argument mismatch; int cannot be converted to capture#1 of ? extends Number)
```

list 는 Number 또는 그 하위 타입을 생산할 수 있지만  
어떤 타입인지 정확히 모르기 때문에 값을 소비(add)할 수는 없다.

`List<Integer>` 인지 `List<Double>`인지 모르니까

반대로

```java
List<? super Number> list = new ArrayList<Object>();
list.add(3);         // ✅ 가능
list.add(3.14);      // ✅ 가능
Object o = list.get(0); // 반환 타입은 Object
```

? super Number는 최소한 Number이거나 그 상위 타입(Object 등)이므로, add는 가능(소비자)

엥 get도 Object로 받았으니까 생산, 소비 다 한 거 아니냐

-> 여기서 get()은 안전하지 않음  
컴파일러는 꺼낸 값이 Number인지 확신할 수 없기 때문

그래서 소비자로 보는게 맞다

---

이번에는 [item 30](https://github.com/column-wise/Effective-Java/blob/main/Chap.05/item_30.md)의 max 함수

```java
public static <E extends Comparable<E>> E max(List<E> list)
```

=>

```java
public static <E extends Comparable<? super E>> E max(List<? extends E> list)
```

일단 list는 E를 생산하므로(read) `<? extends E>`로 수정  
`Comparable<E>`는 E 인스턴스를 소비하여 선후 관계를 뜻하는 정수를 생산한다.

이렇게까지 해야 하나?  
그렇다.

다음 리스트는 오직 수정된 max로만 처리할 수 있다.

```java
List<ScheduledFuture<?>> scheduledFutures = ...;
```

왜냐하면 `ScheduledFuture` 는 `Comparable<ScheduledFuture>`를 구현하지 않았기 때문이다.

대신 ScheduledFuture는 Delayed의 하위 인터페이스인데, Delayed는 `Comparable<Delayed>`를 확장했다.

기존 메서드에서
`<E extends Comparable<E>>`
는 "E는 자기 자신과 비교 가능한 타입이어야 한다"는 뜻인데,

- ScheduledFuture<?>는 Comparable<Delayed>지
- Comparable<ScheduledFuture<?>>가 아님

## 타입 매개변수 VS 와일드카드

타입 매개변수와 와일드카드에는 공통되는 부분이 있어서 메서드를 정의할 때 둘 중 뭘 써도 괜찮은 경우가 있다.

```java
public static <E> void swap(List<E> list, int i, int j);
public static void swap(List<?> list, int i, int j);
```

리스트를 넘기면 i, j 인덱스의 원소를 교환해줄 것인데... 뭐가 나을까

**메서드 선언에 타입 매개변수가 한 번만 나오면 와일드카드로 대체하라**

하지만 두 번째 swap 메서드에는 문제가 있다.

```java
public static void swap(List<?> list, int i, int j) {
    list.set(i, list.set(j, list.get(i)));
}
```

이런 식으로 구현되어 있다면

```
Swap.java:5: error: incompatible types: Object cannot be
converted to CAP#1
    list.set(i, list.set(j, list.get(i)));
                                    ^
    where CAP#1 is a fresh type-variable:
        CAP#1 extends Object from capture of ?
```

이런 오류가 발생하는데, List<?>에는 null 밖에 넣을 수 없기 때문이다.

이것을 형변환이나 리스트의 로 타입을 사용하지 않고 해결하기 위해서는  
와일드카드 타입의 실제 타입을 알려주는 private 도우미 메서드를 작성할 수 있다.

```java
public static void swap(List<?> list, int i, int j) {
    swapHelper(list, i, j);
}

private static <E> void swapHelper(List<E> list, int i, int j) {
    List.set(i, list.set(j, list.get(i)));
}
```

swapHelper는 리스트가 `List<E>`임을 알고 있고, 리스트에서 꺼낸 값은 E 타입임을 알기 때문에 리스트에 다시 넣어도 안전하다.

## 정리

조금 복잡하더라도 와일드카드 타입을 적용하면 API가 훨씬 유연해진다.  
Comparable과 Comparator 모두 소비자라는 사실도 기억하자.
