# item 43. 람다보다는 메서드 참조를 사용하라

람다는 익명 클래스에 비해 간결하다는 아주 큰 장점을 갖는다.

그런데 자바에는 함수 객체를 심지어 람다보다도 더 간결하게 만드는 방법이 있으니,

바로 **메서드 참조(Method Reference)** 다.

다음 코드를 보자.

```java
map.merge(key, 1, (count, incr) -> count + incr);
```

자바 8에 Map에 추가된 merge 메서드를 사용했다.  
merge는 키, 값, 함수를 인수로 받으며, 키가 맵 안에 있다면 키, 값 쌍을 저장하고, 없다면 키, 함수의 결과 값을 저장한다.

🔍 (count, incr) -> count + incr 에서의 두 인자  
count: 기존에 있던 값 (map.get(key))  
incr: merge에 넘긴 값 (1)

깔끔해보이지만 아직 거추장스럽다.

count, incr는 크게 하는 일 없이 공간을 차지한다..  
단순 두 인수의 합을 반환

자바 8부터 Integer 클래스에 이 람다와 같은 기능을 하는 sum이 추가되어

```java
map.merge(key, 1, Integer::sum);
```

보기 좋다.

매개변수 수가 늘어날수록 메서드 참조로 제거할 수 있는 코드양도 늘어난다.

하지만 메서드 참조가 항상 람다보다 나은 것은 아니다.

## 람다가 더 나은 경우

1. 어떤 람다에서는 매개변수의 이름 자체가 힌트가 되므로 메서드 참조보다 길이는 길지만 읽기 쉽고 유지보수가 쉬울 수도 있다.

   ```java
   list.forEach(user -> logger.info("Processing user: {}", user.getName()));
   ```

   ```java
   list.forEach(logger::info); // ❌ 의미 전달 부족
   ```

   무엇을 로그에 찍는지, 문맥상 어떤 로깅인지 알기 어렵다.

2. 람다가 더 짧은 경우(보통 메서드와 람다가 같은 클래스 안에 있는 경우)

   ```java
   public class GoshThisClassNameIsHumongous {

       public void start() {
           service.execute(GoshThisClassNameIsHumongous::action); // 메서드 참조
           service.execute(() -> action());                       // 람다
       }

       public void action() {
           System.out.println("실행 중...");
       }
   }
   ```

   ```java
   service.execute(GoshThisClassNameIsHumongous::action);
   ```

   ```java
   service.execute(() -> action());
   ```

   이런 경우에는 람다 쪽이 더 짧고 명확하다.

## 메서드 참조의 유형

1. 정적 메서드를 가리키는 메서드 참조

2. 수신 객체(Receiving Object; 참조 대상 인스턴스)를 특정하는 한정적(bound) 인스턴스 메서드 참조
   함수 객체가 받는 인수와 참조되는 메서드가 받는 인수가 똑같다

3. 수신 객체를 특정하지 않는 비한정적(unbound) 인스턴스 메서드 참조  
   함수 객체를 적용하는 시점에 수신 객체를 알려준다.  
   이를 위해 수신 객체 전달용 매개변수가 매개변수 목록의 첫 번째로 추가된다.

4. 클래스 생성자

5. 배열 생성자

| 메소드 참조 타입  | Example                  | 같은 기능의 람다                                     |
| ----------------- | ------------------------ | ---------------------------------------------------- |
| Static            | `Integer::parseInt`      | `str -> Integer.parseInt(str)`                       |
| Bound             | `Instant.now()::isAfter` | `Instant then = Instant.now(); t -> then.isAfter(t)` |
| Unbound           | `String::toLowerCase`    | `str -> str.toLowerCase()`                           |
| Class Constructor | `TreeMap<K,V>::new`      | `() -> new TreeMap<K,V>()`                           |
| Array Constructor | `int[]::new`             | `len -> new int[len]`                                |
