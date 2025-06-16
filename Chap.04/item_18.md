# item 18. 상속보다는 컴포지션을 사용하라

상속은 코드를 재사용하는 좋은 수단이지만, 항상 최선은 아니다...  
상위 클래스와 하위 클래스가 같은 개발자가 통제하는 패키지 안 이거나  
확장할 목적으로 설계되었고, 문서화도 잘 된 클래스는 문제가 없겠으나

## 패키지 경계를 넘어 다른 패키지의 구체 클래스를 상속하는 일은 위험하다.

메서드 호출과는 달리 상속은 캡슐화를 깨뜨린다.  
상위 클래스는 릴리스마다 내부 구현이 달라질 수 있으며, 코드 한 줄 수정하지 않은 하위 클래스가 오작동할 수 있기 때문이다.

### 예시

```java
public class InstrumentedHashSet<E> extends HashSet<E> {
    private int addCount = 0;

    @Override public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }
}
```

요소가 몇 번 add 되었는지를 체크하기 위해 HashSet을 상속하고, add, addAll 을 오버라이딩 했다.

```java
InstrumentedHashSet<String> s = new InstrumentedHashSet<>();
s.addAll(List.of("Snap", "Crackle", "Pop"));  // 기대값: 3
System.out.println(s.getAddCount());          // 실제값: 6 ❌
```

addAll 호출로 count가 3 증가했고, 내부적으로 HashSet의 addAll이 루프를 돌며 add()를 3번 호출, count를 한 번씩 더 증가시켜

3개의 원소를 더했지만 6이 됨

**상속이 내부 구현에 의존했기 때문**

`HashSet.addAll()`이 내부적으로 `add()`를 호출한다는 사실은 문서화되어 있지 않은 구현 세부사항이다.

이것은 다음 릴리즈에 수정될 수도 있다.

⚠️ **상위 클래스의 변화가 생기면?**

서브 클래스가 보안이나 검증을 위해 요소 추가 메서드를 모두 오버라이드 해서 필터링 하고 있었다고 해보자

```java
public class SecureVector<E> extends Vector<E> {
    @Override
    public synchronized boolean add(E e) {
        validate(e); // 필터
        return super.add(e);
    }

    @Override
    public synchronized void addElement(E e) {
        validate(e); // 필터
        super.addElement(e);
    }

    private void validate(E e) {
        if (!isValid(e)) throw new SecurityException("Invalid element");
    }
}
```

상위 클래스에 새 메서드가 추가되면???

```java
// hypothetically in a later JDK version
public boolean addIfAbsent(E e) {
    if (!contains(e)) return add(e);  // 기존 add() 호출
    return false;
}
```

이런 식으로 우회하여 원소를 추가할 수 있게 된다...

```java
SecureVector<String> sv = new SecureVector<>();
sv.addIfAbsent("HACKED");  // ✅ addIfAbsent는 오버라이드되지 않음!
```

실제로 `Vector`, `Hashtable` 클래스를 JDK 1.2에서 Collections Framework에 포함할 때 비슷한 보안 이슈가 발생했었음..

## 해결책: 상속 대신 컴포지션 사용

기존 클래스를 확장하는 대신, 새로운 클래스를 만들고 private 필드로 기존 클래스의 인스턴스를 참조하도록 구현

기존 클래스가 새 클래스의 구성요소로 사용된다는 뜻에서 컴포지션(composition; 구성)이라고 한다.

새 클래스의 인스턴스 메서드들은 기존 클래스의 대응하는 메서드를 호출해 그 결과를 반환한다. 이 방식을 전달(forwarding)이라고 한다.

새 클래스의 메서드들은 전달 메서드(forwarding method)라 부른다.

그 결과 새로운 클래스는 기존 클래스의 내부 구현 방식의 영향에서 벗어나며, 기존 클래스에 새로운 메서드가 추가되더라도 전혀 영향 받지 않는다.

### 예시

```java
// Wrapper class - uses composition in place of inheritance
public class InstrumentedSet<E> extends ForwardingSet<E> {
    private int addCount = 0;

    public InstrumentedSet(Set<E> s) {
        super(s);
    }

    @Override public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }
}
```

```java
// Reusable forwarding class
public class ForwardingSet<E> implements Set<E> {
    private final Set<E> s;

    public ForwardingSet(Set<E> s) { this.s = s; }

    public void clear() { s.clear(); }
    public boolean contains(Object o) { return s.contains(o); }
    public boolean isEmpty() { return s.isEmpty(); }
    public int size() { return s.size(); }
    public Iterator<E> iterator() { return s.iterator(); }
    public boolean add(E e) { return s.add(e); }
    public boolean remove(Object o) { return s.remove(o); }
    public boolean containsAll(Collection<?> c) { return s.containsAll(c); }
    public boolean addAll(Collection<? extends E> c) { return s.addAll(c); }
    public boolean removeAll(Collection<?> c) { return s.removeAll(c); }
    public boolean retainAll(Collection<?> c) { return s.retainAll(c); }
    public Object[] toArray() {

```

`ForwardingSet`에서는 생성자에서 받은 Set 인스턴스에 forwarding만 함
`ForwardingSet`을 상속받는 `InstrumentedSet`에서 계측을 담당하고, 실제 데이터 구조 내부 동작은 Set<E> s가 처리함

`InstrumentedSet` 같은 클래스를 래퍼 클래스라고 하며, 다른 Set에 계측 기능을 덧씌운다는 뜻에서 데코레이터 패턴(Decorator pattern) 이라고 한다.

⚠️ **Delegation (위임)**과의 차이점

종종 composition + forwarding을 **delegation(위임)**이라고 부르기도 하지만,
엄밀하게는 다르다.

delegation의 엄밀한 정의:
**래퍼가 자기 자신(this)** 을 피호출 객체에 전달해서,
실제 작업을 피호출 객체가 대신 수행함.

예시:

```java
class EventListener {
private EventHandler handler;

    public EventListener() {
        this.handler = new EventHandler(this); // delegation
    }

}
```

즉, this를 넘기는 구조가 있을 때만 엄밀히 delegation.

### 컴포지션의 단점

콜백 프레임워크와는 어울리지 않는다.

내부 객체는 자신을 감싸고 있는 래퍼의 존재를 모르니 자신(this)을 넘기고, 콜백 때는 래퍼가 아닌 내부 객체를 호출하게 된다.

=> SELF 문제

## 정리

상속은 "is-a" 관계일 때만 적절하다.

"모든 B가 A인가?" 를 만족할 때만 상속, 아니라면 컴포지션을 사용해야 한다.

상속을 잘못 쓰면

1. 구현 세부사항이 API로 노출됨.
   상속을 하면 부모 클래스의 모든 public/protected 메서드가 자식 클래스의 API로 노출됨.

   ex.

   ```java
   Properties props = new Properties();
   props.put("key", "value"); // 사용 가능
   Object val = props.get("key"); // Hashtable에서 상속된 메서드
   ```

   get(String key)는 Hashtable 에서 물려받은 것인데, Properties의 getProperty(String key)와는 다른 동작을 한다. -> 혼란 발생

2. 이상한 동작과 혼란 초래

   ex.

   ```java
   Properties defaults = new Properties();
   defaults.setProperty("language", "Korean");

   Properties props = new Properties(defaults);
   props.setProperty("timezone", "Asia/Seoul");

   // ① default까지 조회
   props.getProperty("language"); // ✅ "Korean"

   // ② 상속된 Hashtable.get은 default 무시
   props.get("language"); // ❌ null
   ```

   props는 defaults의 속성도 담고 있음.  
   getProperty는 defaults 까지 조회하지만, get은 그렇지 않음.
   동일한 key에 대해 다르게 동작하는 두 메서드 존재

   Hashtable의 모든 메서드가 그대로 열려 있으므로, 사용자는 혼동하거나 잘못 사용할 여지가 있음.

3. 불변식(invariant) 깨짐

   Hashtable은 타입 제한이 없음.

   ```java
   public V put(K key, V value); // K, V는 Object → 어떤 타입이든 가능
   ```

   `Properties` 클래스는 String key와 String value만 저장하도록 설계됨.
   그리고 Properties는 put()을 오버라이드 하지 않았기 때문에 Hashtable의 put()을 그대로 사용

   ```java
   Properties props = new Properties();
   props.put(42, true); // key: Integer, value: Boolean

   // 이후 저장 시도
   FileOutputStream fos = new FileOutputStream("out.properties");
   props.store(fos, "test");
   ```

   -> 결과

   ```
    java.lang.ClassCastException: java.lang.Integer cannot be cast to java.lang.String
   ```

   불변식(String only)이 깨짐.......
