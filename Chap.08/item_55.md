# item 55. 옵셔널 반환은 신중히 하라

자바 8 전에는 특정 조건에서 값을 반환할 수 없을 때

1. 예외를 던지거나
2. null을 반환

하도록 구현해야 했다.

하지만 예외는 진짜 예외적인 상황에서만 사용해야 하며,  
예외를 생성할 때 스택 추적 전체를 캡쳐하므로 무거운 작업이다.

null을 반환할 수 있는 메서드를 호출할 때는 별도의 null 처리 코드를 추가해줘야 한다.

자바 8부터는 Optional\<T>을 반환할 수 있는 선택지가 생겼는데,

예외를 던지는 메서드보다 유연하고, 사용하기 쉬우며, null을 반환하는 메서드보다 오류 가능성이 적다.

ex. 컬렉션에서 최댓값을 구하는 함수

```java
// Returns maximum value in collection - throws exception if empty
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

item 30에서 본 이 메서드는 컬렉션이 비어있을 때 예외를 던진다.

Optional을 던지도록 수정하면

```java
// Returns maximum value in collection as an Optional<E>
public static <E extends Comparable<E>> Optional<E> max(Collection<E> c) {
    if (c.isEmpty())
        return Optional.empty();

    E result = null;

    for (E e : c)
        if (result == null || e.compareTo(result) > 0)
            result = Objects.requireNonNull(e);

    return Optional.of(result);
}
```

보다시피 옵셔널을 반환하도록 구현하는 것은 어렵지 않다.  
적절한 정적 팩토리(Optional.empty(), Optional.of() 등)를 이용하면 된다.

---

Optional을 사용하는 이유

API 사용자에게 반환값이 없을 수도 있다는 것을 명확히 전달  
비검사 예외를 던지거나 null을 반환한다면 API 사용자가 그 사실을 인지하지 못해 에러가 발생할 수 있다.

클라이언트가 옵셔널을 받았다면 Optional이 비어있을 때 취할 행동을 선택해야 하는데

1. 기본값 지정: orElse()

   ```java
   String lastWordInLexicon = max(words).orElse("No words...");
   ```

   기본값 계산 비용이 적을 때 사용

2. 예외 던지기: orElseThrow()

   ```java
   Toy myToy = max(toys).orElseThrow(TemperTantrumException::new);
   ```

   실제 예외가 아니라 예외 팩토리를 넘겨 예외가 실제로 발생하지 않는 상황에서는 예외 생성 비용이 들지 않는다.

3. 무조건 값이 있다고 가정: get()

   ```java
   Element lastNobleGas = max(Elements.NOBLE_GASES).get();
   ```

   옵셔널에 값이 항상 있다고 판단한다면 바로 값을 꺼내 사용할 수도 있지만  
   만약 비어있다면 `NoSuchElementException` 발생

4. 지연 평가 기본 값 설정: orElseGet(Supplier\<T>)

   ```java
   String word = max(words).orElseGet(() -> computeExpensiveDefault());
   ```

   기본값 계산 비용이 클 때 사용하면 값이 처음 필요할 때 Supplier를 사용해 생성하므로 초기 설정 비용을 낮출 수 있다.

5. 그 밖에 Optional 고급 메서드

   | 메서드                                           | 설명                                           |
   | ------------------------------------------------ | ---------------------------------------------- |
   | `filter(Predicate)`                              | 조건을 만족하지 않으면 비어있는 Optional 반환  |
   | `map(Function)`                                  | 값을 변환 (값이 없으면 비어있는 Optional 유지) |
   | `flatMap(Function)`                              | Optional을 중첩하지 않고 평탄화                |
   | `ifPresent(Consumer)`                            | 값이 있으면 처리 수행                          |
   | `or(Supplier<Optional>)` _(Java 9)_              | 대체 Optional 제공                             |
   | `ifPresentOrElse(Consumer, Runnable)` _(Java 9)_ | 값 유무에 따라 각각 처리                       |

---

Optional을 사용하지 말아야 할 상황

1. 컨테이너 타입을 감싸는 경우

   Optional<List\<T>>, Optional<String[]> 대신 빈 리스트, 빈 배열을 반환하는게 좋다(Item 54)

2. 성능이 중요한 경우

   Optional도 엄연히 새로 할당하고 초기화해야 하는 객체이고, 값을 꺼내려면 메서드를 호출해야 하므로 한 단계를 더 거침.

   특히 박싱된 기본 타입을 담는 Optional(Optional\<Integer> 등)은 값을 두 번이나 감싸기 때문에 특히 무겁다.

   그래서 int, long, double 전용 OptionalInt, OptionlLong, OptionalDouble 클래스도 있으므로 박싱된 기본 타입을 담은 옵셔널은 사용하지 말자.

3. 옵셔널을 컬렉션의 키, 값, 원소나 배열의 원소로 사용하지 말자
   대부분의 경우에서 적절하지 않다.
