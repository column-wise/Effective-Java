# item 44. 표준 함수형 인터페이스를 사용하라

자바가 람다를 지원하면서 API를 작성하는 모범 사례도 크게 바뀌었다.

예를 들어 상위 클래스의 기본 메서드를 재정의해 원하는 동작을 구현하는 템플릿 메서드 패턴의 매력이 크게 줄었다.  
대신 같은 효과의 함수 객체를 받는 정적 팩토리나 생성자를 제공할 수 있다.

ex.

```java
// 템플릿 메서드 패턴
abstract class Processor {
    public final void process() {
        before();
        doWork();
        after();
    }

    protected void before() {}
    protected abstract void doWork();
    protected void after() {}
}
```

=>

```java
public class Processor {
    public void process(Runnable task) {
        // 공통 처리
        System.out.println("Before");
        task.run(); // 동작은 함수 객체로 대체
        System.out.println("After");
    }
}
```

```java
processor.process(() -> System.out.println("Doing work"));
```

함수 객체를 매개변수로 받는 생성자와 메서드를 만들 때,  
함수형 매개변수 타입을 올바르게 선택해야 한다.

LinkedHashMap을 생각해보자....  
이 클래스의 protected 메서드인 removeEldestEntry를 재정의하면 캐시로 사용할 수 있다.

```java
protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
    return size() > 100;
}
```

removeEldestEntry는 size()를 호출해 맵 안의 원소 수를 알아내는데,  
removeEldestEntry가 인스턴스 메서드라 가능한 방식이다.

하지만 생성자에 넘기는 함수 객체는 이 맵의 인스턴스 메서드가 아니다.  
팩토리나 생성자를 호출할 때는 맵의 인스턴스가 없기 때문이다.

이를 반영해서 함수형 인터페이스를 만들면

```java
// Unnecessary functional interface; use a standard one instead.
@FunctionalInterface interface EldestEntryRemovalFunction<K,V>{
    boolean remove(Map<K,V> map, Map.Entry<K,V> eldest);
}
```

동작하기는 하지만 굳이 이용할 이유가 없다.

java.util.function 패키지를 보면 이미 다양한 용도의 표준 함수형 인터페이스가 정의되어 있다.

직접 만든 EldestEntryRemovalFunction 대신 BiPredicate<Map<K,V>, Map.Entry<K,V>>를 사용할 수 있다.

## 표준 함수형 인터페이스

java.util.function 패키지에는 43개의 인터페이스가 담겨있다.

1.  기본형

    | Interface         | Function Signature  | Example               |
    | ----------------- | ------------------- | --------------------- |
    | UnaryOperator<T>  | T apply(T t)        | `String::toLowerCase` |
    | BinaryOperator<T> | T apply(T t1, T t2) | `BigInteger::add`     |
    | Predicate<T>      | boolean test(T t)   | `Collection::isEmpty` |
    | Function<T, R>    | R apply(T t)        | `Arrays::asList`      |
    | Supplier<T>       | T get()             | `Instant::now`        |
    | Consumer<T>       | void accept(T t)    | `System.out::println` |

2.  기본형(primitive) 전용 변형 – int, long, double 지원  
    🟦 Predicate

        IntPredicate

        LongPredicate

        DoublePredicate

    🟦 Consumer

        IntConsumer

        LongConsumer

        DoubleConsumer

    🟦 Supplier

        BooleanSupplier ← 유일하게 boolean 반환

    🟦 Function

        기본형 반환: IntFunction<R>, LongFunction<R>, DoubleFunction<R>

        기본형 입력, 참조형 반환: IntToObjFunction<R>, LongToObjFunction<R>, DoubleToObjFunction<R>

        기본형 → 기본형: IntToDoubleFunction, LongToIntFunction, DoubleToIntFunction 등 (총 6개)

    → 총 9개의 Function 변형

3.  2-인자 함수형 인터페이스(BiXxx)
    🟨 참조형 두 개 받는 인터페이스

        BiPredicate<T, U>

        BiFunction<T, U, R>

        BiConsumer<T, U>

    🟨 참조형 2개 + 기본형 반환

        ToIntBiFunction<T, U>

        ToLongBiFunction<T, U>

        ToDoubleBiFunction<T, U>

    🟨 참조형 + 기본형 소비자

        ObjIntConsumer<T>

        ObjLongConsumer<T>

        ObjDoubleConsumer<T>

    → 총 9개의 BiXxx 인터페이스

## 표준 인터페이스 중 필요한 용도에 맞는 게 없다면 직접 작성

Comparator\<T> 는 (T, T) -> int 로 동작하므로 ToIntBiFunction\<T, T> 와 동일하다.

하지만 Comparator는

1. 의미 있는 이름을 제공한다.
2. 구현하는 쪽에서 반드시 지켜야 할 규약을 담고있다.
3. 기본 메서드가 많다.

   thenComparing(), reversed(), comparing() 등 조합 가능한 기본 메서드들이 포함되어 있다.

Comparator 처럼 전용 함수형 인터페이스로 구현해도 좋은 경우는 다음 조건 중 하나 이상을 만족하는 경우이다.

1. 자주 쓰이며, 이름 자체가 용도를 명확히 설명해준다.
2. 반드시 따라야 하는 규약이 있다.
3. 유용한 디폴트 메서드를 제공할 수 있다.

전용 함수형 인터페이스를 구현했다면,

1. 해당 클래스의 코드나 설명 문서를 읽을 이에게 이 인터페이스가 람다용으로 설계된 것임을 알린다.
2. 해당 인터페이스가 추상 메서드를 오직 하나만 가지고 있어야 컴파일되게 해준다.
3. 유지보수 과정에서 실수로 메서드를 추가하지 못하도록 막는다.

그리고 직접 만든 함수형 인터페이스에는 `@FunctionalInterface` 어노테이션을 추가하라

## 함수형 인터페이스를 API에서 사용할 때의 주의점

같은 메서드에 대해 서로 다른 함수형 인터페이스 타입을 받는 여러 오버로딩을 제공하지 마라

ex. ExecutorService.submit(...)

```java
<T> Future<T> submit(Callable<T> task);
Future<?> submit(Runnable task);
```

둘 다 submit(() -> ...) 식으로 람다를 전달받을 수 있지만, 람다는 Runnable도 되고 Callable도 될 수 있으므로 모호하다.

결과적으로 명시적 캐스팅이 필요하게 된다.

```java
executor.submit((Callable<Void>) () -> {
    // do something
    return null;
});
```
