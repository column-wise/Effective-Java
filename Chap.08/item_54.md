# item 54. null이 아닌, 빈 컬렉션이나 배열을 반환하라

ex. 컬렉션이 비어있을 때 null을 반환하는 메서드

```java
private final List<Cheese> cheesesInStock = ...;

/**
* @return a list containing all of the cheeses in the shop,
* or null if no cheeses are available for purchase.
*/
public List<Cheese> getCheeses() {
    return cheesesInStock.isEmpty() ? null
        : new ArrayList<>(cheesesInStock);
}
```

흔하게 볼 수 있는 메서드다.  
하지만 이처럼 null을 반환하게 되면

클라이언트 쪽에서는

```java
List<Cheese> cheeses = shop.getCheeses();
if (cheeses != null && cheeses.contains(Cheese.STILTON))
    System.out.println("Jolly good, just the thing.");
```

같은 방어코드를 항상 넣어줘야 한다.

빈 컨테이너를 할당하는데에도 비용이 드니 null을 반환하는 것이 낫다는 주장도 있는데,

1. 그 정도의 성능 차이는 신경 쓸 수준이 못된다.

2. 빈 컬렉션과 배열은 굳이 새로 할당하지 않고도 반환할 수 있다.

```java
//The right way to return a possibly empty collection
public List<Cheese> getCheeses() {
    return new ArrayList<>(cheesesInStock);
}
```

---

만약 빈 컬렉션 할당이 성능을 떨어뜨리는 특수한 상황이라면,

빈 불변 객체를 반환할 수 있다.

```java
// Optimization - avoids allocating empty collections
public List<Cheese> getCheeses() {
    return cheesesInStock.isEmpty() ? Collections.emptyList()
        : new ArrayList<>(cheesesInStock);
}
```

불변 객체는 자유롭게 공유해도 안전하므로  
Collections.emptyList, Collections.emptySet, Collections.emptyMap 등을 사용할 수 있다.

단, 이 역시 최적화에 해당하니 꼭 필요할 때만 사용하자.

> Premature optimization is the root of all evil.  
> Donald Knuth (컴퓨터 과학자)

성능 병목이 없는데도 불구하고, 복잡한 최적화를 미리 적용하는 것은  
코드를 읽기 어렵게 만들고, 버그 발생 가능성을 높이므로

---

배열의 경우도 마찬가지다.

null을 반환하지 말고 길이가 0인 배열을 반환하라.

```java
//The right way to return a possibly empty array
public Cheese[] getCheeses() {
    return cheesesInStock.toArray(new Cheese[0]);
}
```

※ toArray의 파라미터(길이 0의 배열)는 반환타입을 알려주는 역할을 한다.

다음 방법처럼 미리 길이 0의 배열을 선언해두고 그 배열을 반환해도 된다.

```java
// Optimization - avoids allocating empty arrays
private static final Cheese[] EMPTY_CHEESE_ARRAY = new Cheese[0];
public Cheese[] getCheeses() {
    return cheesesInStock.toArray(EMPTY_CHEESE_ARRAY);
}
```

길이가 0인 배열은 불면이니까
