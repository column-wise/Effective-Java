# item 33. 타입 안전 이종 컨테이너를 고려하라

제네릭은 `Set<E>`, `Map<K, V>` 등의 컬렉션과 `ThreadLocal<T>`, `AtomicReference<T>` 등의 단일 원소 컨테이너에도 흔히 쓰인다.

하나의 컨테이너에서 매개변수화 할 수 있는 타입의 수가 제한된다.  
예를 들어 Set에는 원소의 타입을 뜻하는 단 하나의 타입 매개변수,  
Map에는 키와 값의 타입을 뜻하는 2개만 필요한 식이다.

하지만 더 유연한 수단이 필요할 때도 종종 있다.  
예를 들어 데이터베이스의 모든 열을 타입 안전하게 이용하고 싶은 상황

ex.

```java
Object o = map.get("age");
int age = (Integer) o;  // 형변환 필요, 안전하지 않음

o = map.get("name");
String name = (String) o;
```

컨테이너 대신 키를 매개변수화한 다음, 컨테이너에 값을 넣거나 뺄 때 매개변수화 한 키를 함께 제공하면 된다. (?)

이러한 설계 방식을 타입 안전 이종 컨테이너 패턴이라고 한다.

## 타입 안전 이종 컨테이너 패턴 (type safe heterogeneous container pattern)

타입별로 즐겨 찾는 인스턴스를 저장하고 검색할 수 있는 Favorites 클래스를 생각해보자.  
각 타입의 Class 객체를 매개변수화한 키 역할로 사용

class 리터럴의 타입은 Class가 아닌 Class\<T> 이다.

```java
// Typesafe heterogeneous container pattern - API
public class Favorites {
    public <T> void putFavorite(Class<T> type, T instance);
    public <T> T getFavorite(Class<T> type);
}
```

```java
// Typesafe heterogeneous container pattern - client
public static void main(String[] args) {
    Favorites f = new Favorites();
    f.putFavorite(String.class, "Java");
    f.putFavorite(Integer.class, 0xcafebabe);
    f.putFavorite(Class.class, Favorites.class);

    String favoriteString = f.getFavorite(String.class);
    int favoriteInteger = f.getFavorite(Integer.class);
    Class<?> favoriteClass = f.getFavorite(Class.class);

    System.out.printf("%s %x %s%n", favoriteString,
    favoriteInteger, favoriteClass.getName());
}
```

Favorites 인스턴스는 타입 안전하다. String을 요청했는데 Integer를 반환하는 일은 없다.

```java
//Favorites 클래스 구현
public class Favorites {
    private Map<Class<?>, Object> favorites = new HashMap<>();

    public <T> void putFavorite(Class<T> type, T instance) {
        favorites.put(Objects.requireNonNull(type), instance);
    }

    public <T> T getFavorite(Class<T> type) {
        return type.cast(favorites.get(type));
    }
}
```

Map의 키가 `Class<?>` 라 null 외에 아무것도 넣을 수 없다고 생각할 수 있지만,  
맵이 아니라 키가 와일드카드 타입인 것이다.  
모든 키가 서로 다른 매개변수화 타입일 수 있다.

또한 favorites 맵의 값 타입은 Object로, 이 맵은 키와 값 사이의 타입 관계를 보증하지 않는다.

getFavorites 에서 값을 꺼내면 Object 타입이므로 형변환을 해줘야 한다.

여기서 사용되는 cast 메서드는 형변환 연산자의 동적 버전이다.  
이 메서드는 단순히 주어진 인수가 Class 객체가 알려주는 타입의 인스턴스인지를 검사한 다음, 맞다면 그대로 반환하고, 아니면 ClassCastException을 던진다.

```java
public class Class<T> {
    T cast(Object obj);
}
```

그런데 cast 메서드가 단지 인수를 그대로 반환하기만 한다면 굳이 왜 사용하는 것일까?

cast 메서드의 시그니처가 Class 클래스가 제네릭이라는 이점을 완벽히 활용하기 때문이다.  
cast의 반환 타입은 Class 객체의 타입 매개변수와 같다.

다만, 악의적인 클라이언트가 Class 객체를 제네릭이 아닌 로타입으로 넘기면 Favorites 인스턴스의 타입 안정성이 쉽게 깨진다.

-> 동적 형변환을 쓰면 된다.

```java
// Achieving runtime type safety with a dynamic cast
public <T> void putFavorite(Class<T> type, T instance) {
    favorites.put(type, type.cast(instance));
}
```

또한, Favorites는 실체화 불가 타입을 다룰 수 없다.  
String, String[] 은 저장할 수 있지만, List\<String>은 저장할 수 없다.

Class\<T> 자리에 Class.List\<String> 라고 쓰면 오류가 날 것이고, List\<String>, List\<Integer> 모두 List.class 라는 Class 객체를 공유하므로 둘 다 허용해서 똑같은 타입의 객체 참조를 반환한다면 Favorites 객체의 내부는 아수라장이 될 것이다.

※ 슈퍼 타입 토큰(super type token) 이라는 우회법이 있으나 완벽하지 않다.

---

Favorites가 사용하는 타입 토큰(Class\<T>)은 비한정적이다.

이런 타입 토큰에 타입 제한을 걸 수 있다.

```java
public <T extends Annotation>
    T getAnnotation(Class<T> annotationType);
```

Annotation의 하위타입만 넣어라 라는 제한을 건 타입 토큰

Class\<?> 타입의 객체가 있고, 이를 getAnnotation 처럼 한정적 타입 토큰을 받는 메서드에 넘기려면 어떻게 해야 할까?

객체를 Class<? extends Annotation> 으로 형변환할 수 있겠지만, 이 형변환은 비검사이므로 경고가 뜰 것이다.

Class.asSubclass 메서드를 사용하면 호출된 인스턴스 자신의 Class 객체를 인수가 명시한 클래스로 형변환 한다.

```java
// Use of asSubclass to safely cast to a bounded type token
static Annotation getAnnotation(AnnotatedElement element, String annotationTypeName) {
    Class<?> annotationType = null; // Unbounded type token

    try {
        annotationType = Class.forName(annotationTypeName);
    } catch (Exception ex) {
        throw new IllegalArgumentException(ex);
    }

    return element.getAnnotation(annotationType.asSubclass(Annotation.class));
}
```

annotationType을 Annotation 클래스로 형변환해서 리턴  
컴파일 시점에는 타입을 알 수 없는 인스턴스를 런타임에 클래스를 읽어냄
