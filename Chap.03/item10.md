# item 10. equals는 일반 규약을 지켜 재정의하라

## equals 재정의를 피해야 할 때

equals 메서드는 재정의하기 쉬워 보이지만, 그렇지 않다.  
아예 재정의하지 않는 것도 방법인데, 그렇게 하면 그 클래스의 인스턴스는 오직 자기 자신과만 같게 된다.

아래 상황 중 하나에 해당한다면 재정의하지 않는 것이 최선이다.

1. 각 인스턴스가 본질적으로 고유하다.

   값을 표현하는 것이 아닌, 동작하는 개체를 표현하는 클래스가 여기에 해당한다.  
   예를 들어 `Thread`, `Socket`, `FileInputStream`, `DatabaseConnection` 클래스 등이 있다.

   쓰레드가 같은 내부 코드를 가지고 있더라도 서로 다른 작업 단위이므로 == 동일성 비교를 사용하는 것이 맞다.

2. 인스턴스의 '논리적 동치성(Logical Equality)'을 검사할 일이 없다.

   `java.util.regex.Pattern`을 예로 들면,  
   `Pattern`의 equals는 재정의되어 있지 않기 때문에

   ```java
   Pattern p1 = Pattern.compile("\\d+");
   Pattern p2 = Pattern.compile("\\d+");
   System.out.println(p1.equals(p2)); // false
   ```

   위와 같은 결과가 나오는데,

   ```java
   @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof LogicalPattern)) return false;
        LogicalPattern that = (LogicalPattern) o;
        return Objects.equals(this.regex, that.regex);
    }

    @Override
    public int hashCode() {
        return Objects.hash(regex);
    }
   ```

   만약 `Pattern`의 equals, hashCode를 위처럼 재정의되어 있다면, equals는 두 `Pattern`의 인스턴스가 같은 정규표현식을 나타내는지를 검사, 즉 논리적 동치성을 검사하여 p1.equals(p2)의 결과는 true가 될 것이다.

   하지만 대부분의 사용자(클라이언트)는 Pattern 인스턴스를 비교하지 않기 때문에  
   설계자는 이 방식이 애초에 필요하지 않다고 판단한 것이다.  
   이런 경우라면 Object의 기본 equals만으로 해결된다.

3. 상위 클래스에서 재정의한 equals가 하위 클래스에도 딱 들어맞는다.

   대부분의 `Set` 구현체는 `AbstractSet`이 구현한 eqauls를 상속받아 쓰고,  
   `List` 구현체는 `AbstractList`로부터, `Map` 구현체들은 `AbstractMap`으로부터 상속받아 그대로 쓴다.

4. 클래스가 private이거나 package-private이고 equals 메서드를 호출할 일이 없다.

   외부에서 직접 사용하거나 비교할 일이 없다.  
   equals가 실수로라도 호출되는 일을 피하고 싶다면

   ```java
   @Override
   public boolean equals(Object o) {
      throw new AssertionError();   // 호출 금지!
   }
   ```

   처럼 구현해두자.

## equals를 재정의해야 할 때

객체 식별성(Object Identity; 두 객체가 물리적으로 같은가)이 아닌
논리적 동치성을 확인해야 할 때 equals를 재정의한다.

주로 값 클래스(String, Integer 처럼 값을 표현하는 클래스)들이 이런 경우에 해당

두 값 객체를 equals로 비교하려는 프로그래머는 객체가 물리적으로 같은지가 아니라 값이 같은지를 알고 싶어할 것이다.  
equals가 논리적 동치성을 확인하도록 재정의하면, 그 인스턴스는 값을 비교하길 원하는 기대에 부응할 뿐만 아니라 `Map`의 키와 `Set`의 원소로 사용할 수 있게 된다.

값 클래스라고 해도, 값이 같은 인스턴스가 둘 이상 만들어지지 않음을 보장하는 인스턴스 통제 클래스라면 equals를 재정의하지 않아도 된다.  
어차피 논리적으로 같은 인스턴스가 2개 이상 만들어지지 않으니 논리적 동치성과 객체 식별성이 같은 의미가 된다.

## equals 메서드 재정의할 때 지켜야할 일반 규약

다음은 `Object` 명세에 적힌 규약이다.

> equals 메서드는 동치관계(Equivalence Relation)를 구현하며, 다음을 만족한다.

> - 반사성(Reflexivity): null이 아닌 모든 참조 값 x에 대해, x.equals(x)는 true
> - 대칭성(Symmetry): null이 아닌 모든 참조 값 x, y에 대해, x.equals(y)가 true이면, y.equals(x)도 true다.
> - 추이성(Transitivity): null이 아닌 모든 참조 값 x, y, z에 대해, x.equals(y)가 true이고, y.equals(z)도 true면, x.equals(z)도 true다.
> - 일관성(Consistency): null이 아닌 모든 참조 값 x, y에 대해 x.equals(y)를 반복해서 호출하면 항상 true를 반환하거나 항상 false를 반환한다.
> - null-아님: null이 아닌 모든 참조 값 x에 대해, x.equals(null)은 false다.

여기서 말하는 동치관계란 무엇일까?  
집합을 서로 같은 것으로 여겨지는 원소들끼리 묶은 부분집합으로 나누는 관계이다.

이 부분집합을 동치류(Equivalence Class; 동치 클래스)라고 한다.

```
집합 A = {1, 2, 3, 4, 5, 6}
동치 관계: "짝수끼리는 같고, 홀수끼리는 같다"는 기준
→ 동치류:
  - {1, 3, 5}
  - {2, 4, 6}
```

equals 메서드가 쓸모 있으려면 모든 원소가 같은 동치류에 속한 어떤 원소와도 서로 교환할 수 있어야 한다.

---

이번에는 동치관계를 만족시키기 위한 조건 다섯 가지를 살펴보자.

### 반사성

반사성은 단순히 말하면 객체는 자기 자신과 같아야 한다는 뜻이다.  
이 요건은 일부러 어기는 경우가 아니라면 만족시키지 못하기가 더 어려워 보인다.

### 대칭성

대칭성은 두 객체는 서로에 대한 동치 여부에 똑같이 답해야 한다는 뜻이다.  
반사성 요건과 달리 대칭성 요건은 자칫하면 어길 수 있어 보인다.

대소문자를 구별하지 않는 문자열을 구현한 다음 클래스를 살펴보자.  
이 클래스에서 toString 메서든느 원본 문자열의 대소문자를 그대로 돌려주지만 equals에서는 대소문자를 무시한다.

```java
// Broken - violates symmetry!
public final class CaseInsensitiveString {
    private final String s;

    public CaseInsensitiveString(String s) {
        this.s = Objects.requireNonNull(s);
    }

    // Broken - violates symmetry!
    @Override
    public boolean equals(Object o) {
        if (o instanceof CaseInsensitiveString)
            return s.equalsIgnoreCase(((CaseInsensitiveString) o).s);

        if (o instanceof String) // One-way interoperability!
            return s.equalsIgnoreCase((String) o);

        return false;
    }

    // ... Remainder omitted
}
```

```java
CaseInsensitiveString cis = new CaseInsensitiveString("Hello");
String s = "hello";

System.out.println(cis.equals(s)); // true
System.out.println(s.equals(cis)); // false !! (String은 CaseInsensitiveString을 모름)
```

s.equals는 cis를 `CaseInsensitiveString`의 존재를 모르므로 String으로 판단, s.equals(cis)는 false를 반환하여 대칭성을 위반한다.

이번에는 `CaseInsensitiveString`을 컬렉션에 넣어보자.

```java
List<CaseInsensitiveString> list = new ArrayList<>();
list.add(cis)
```

list.contains(s)를 호출하면 결과값이 어떻게 될까?  
현재의 OpenJDK에서는 false를 반환하기는 하지만, 구현에 달렸기 때문에 OpenJDK 버전이 바뀌거나 다른 JDK에서는 true를 반환하거나 런타임 예외를 던질 수도 있다.

equals 규약을 어기면 그 객체를 사용하는 다른 객체들이 어떻게 반응할지 알 수 없다.

결국 `CaseInsensitiveString`의 equals를 String과 호환하겠다는 목표는 달성할 수 없다.

### 추이성

추이성은 첫 번째 객체와 두 번째 객체가 같고, 두 번째 객체와 세 번째 객체가 같다면, 첫 번째 객체와 세 번째 객체도 같아야 한다는 것이다.

이 요건도 간단하지만 어기기 쉽다.

상위 클래스에는 없는 새로운 필드를 하위 클래스에 추가하는 상황을 생각해보자.

```java
// Point 클래스
public class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Point))
            return false;
        Point p = (Point) o;
        return p.x == x && p.y == y;
    }

    // ... Remainder omitted
}
```

```java
// ColorPoint 클래스
public class ColorPoint extends Point {
    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        super(x, y);
        this.color = color;
    }

    // ... Remainder omitted
}
```

`equals`를 재정의하지 않는다면 색상 정보는 무시한 채 비교를 수행하게 된다.

다음처럼 비교 대상이 또 다른 `ColorPoint`고 위치와 색상이 같을 때만 true를 반환하는 `equals`를 생각해보자.

```java

    // Broken - violates symmetry!
    @Override
    public boolean `equals`(Object o) {
        if (!(o instanceof ColorPoint))
            return false;
        return super.`equals`(o) && ((ColorPoint) o).color == color;
    }
```

이 메서드는 일반 `Point`를 `ColorPoint`에 비교한 결과와 그 둘을 바꿔 비교한 결과가 다를 수 있다...(대칭성 위반)

`Point`의 `equals`는 색상을 무시하고, `ColorPoint`의 `equals`는 입력 매개변수의 클래스 종류가 다르다며 매번 false를 반환할 것이다..

```java
Point p = new Point(1, 2);
ColorPoint cp = new ColorPoint(1, 2, Color.RED);

p.`equals`(cp);  // true → Point의 `equals`는 ColorPoint의 color 무시
cp.`equals`(p);  // false → ColorPoint의 `equals`는 color까지 비교함
```

그렇다면 `ColorPoint.equals`가 `Point`와 비교할 때는 색상을 무시하도록 하면 어떨까?

```java
// Broken - violates transitivity!
@Override
public boolean equals(Object o) {
    if (!(o instanceof Point))
        return false;

    // If o is a normal Point, do a color-blind comparison
    if (!(o instanceof ColorPoint))
        return o.equals(this);

    // o is a ColorPoint; do a full comparison
    return super.equals(o) && ((ColorPoint) o).color == color;
}
```

이번에는 추이성(a == b, b == c -> a == c)을 위반한다...

```java
ColorPoint p1 = new ColorPoint(1, 2, Color.RED);
Point p2 = new Point(1, 2);
ColorPoint p3 = new ColorPoint(1, 2, Color.BLUE);

// 각 비교 결과
p1.equals(p2); // true - 색상을 무시하고 좌표만 비교
p2.equals(p3); // true - 역시 색상 무시
p1.equals(p3); // false - 같은 좌표지만 색상이 다름
```

또한 이 방식은 무한 재귀에 빠질 위험도 있다.  
Point의 또 다른 하위 클래스로 `SmellPoint`를 만들고, equals는 같은 방식으로 구현했다고 해보자.

(이부분)

```java
if (!(o instanceof ColorPoint))
    return o.equals(this); // 상대방에게 위임
```

그런 다음 myColorPoint.equals(mySmellPoint)를 호출하면 `StackOverflowError`를 일으킨다.

```java
ColorPoint cp = new ColorPoint(1, 2, Color.RED);
SmellPoint sp = new SmellPoint(1, 2, "mint");

cp.equals(sp)
→ sp.equals(cp)
→ cp.equals(sp)
→ sp.equals(cp)
→ ... 무한 반복 → StackOverflowError!
```

그렇다면 해법은 무엇일까...??

사실 이 현상은 모든 객체 지향 언어의 동치관계에서 나타나는 근본적인 문제이다.  
구체 클래스를 확장해 새로운 값을 추가하면서 equals 규약을 만족시킬 방법은 존재하지 않는다(!)

객체 지향적 추상화의 이점을 포기하지 않는 한은 말이다.  
-> 오잉? 그렇다면 equals 안의 instanceof 검살르 getClass 검사로 고치면 규약도 지키고 값도 추가하면서 구체 클래스를 상속할 수 있지는 않을까??

->

```java
// Broken - violates Liskov substitution principle (page 59)
@Override public boolean equals(Object o) {
   if (o == null || o.getClass() != getClass())
      return false;

   Point p = (Point) o;
   return p.x == x && p.y == y;
}
```

이번 equals는 같은 구현 클래스의 객체와 비교할 때만 true를 반환한다.  
괜찮아보이지만 실제로 사용할 수는 없다.

Point의 하위 클래스는 정의상 여전히 Point 이므로 어디서든 Point로써 활용될 수 있어야 한다.

예를 들어 주어진 점이 반지름이 1인 단위 원 위에 있는지를 판별하는 메서드가 필요하다고 해보자.

```java
// Initialize unitCircle to contain all Points on the unit circle
private static final Set<Point> unitCircle = Set.of(
      new Point( 1, 0), new Point( 0, 1),
      new Point(-1, 0), new Point( 0, -1));

public static boolean onUnitCircle(Point p) {
   return unitCircle.contains(p);
}
```

이제 필드를 추가하지 않는 방식으로 `Point`를 확장해보겠다.

```java
public class CounterPoint extends Point {
    private static final AtomicInteger counter = new AtomicInteger();

    public CounterPoint(int x, int y) {
        super(x, y);
        counter.incrementAndGet();
    }
    public static int numberCreated() { return counter.get(); }
}
```

만들어진 인스턴스의 개수를 생성자에서 count하는 클래스이다.

리스코프 치환 원칙(Liskov Substitution Principle)에 따르면, 어떤 타입에 있어 중요한 속성이라면 그 하위 타입에서도 마찬가지로 중요하다.  
따라서 그 타입의 모든 메서드가 하위 타입에서도 똑같이 잘 작동해야 한다.

그런데 `CounterPoint`의 인스턴스를 `onUnitCircle` 메서드에 넘기면 어떻게 될까?

`Point` 클래스의 `equals`를 `getClass`를 이용해 구현했다면, `onUnitCircle`은 `ConterPoint` 인스턴스의 x, y 값에 상관 없이 false를 반환할 것이다.

컬렉션 구현체에서 주어진 원소를 담고 있는지를 확인할 때 `equals` 메서드를 이용하는데, `CounterPoint`의 인스턴스는 어떤 `Point` 와도 같을 수 없기 때문이다.

반면, `Point`의 `equals`를 `instanceof` 기반으로 올바르게 구현했다면 `CounterPoint` 인스턴스를 건네줘도 `onUnitCircle` 메서드가 제대로 동작할 것이다.

그렇다면 구체 클래스의 하위 클래스에서 값을 추가하면서 `equals`가 동작하게 만들 우회법은 없는 것일까?

상속 대신 컴포지션을 사용하면 된다.(item 18)

`Point`를 상속하는 대신 `Point`를 `ColorPoint`의 private 필드로 두고, `ColorPoint`와 같은 위치의 일반 `Point`를 반환하는 뷰 메서드(item 6)를 public으로 추가하는 식이다.

```java
// Adds a value component without violating the equals contract
public class ColorPoint {
    private final Point point;
    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        point = new Point(x, y);
        this.color = Objects.requireNonNull(color);
    }

    /**
     * Returns the point-view of this color point.
     */
    public Point asPoint() {
        return point;
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof ColorPoint))
            return false;
        ColorPoint cp = (ColorPoint) o;
        return cp.point.equals(point) && cp.color.equals(color);
    }

    // ... Remainder omitted
}
```

```java
ColorPoint cp1 = new ColorPoint(1, 2, Color.RED);
ColorPoint cp2 = new ColorPoint(1, 2, Color.RED);
ColorPoint cp3 = new ColorPoint(1, 2, Color.BLUE);

cp1.equals(cp2); // true
cp1.equals(cp3); // false
```

`equals`는 오직 같은 `ColorPoint` 끼리만 비교하도록 하여 대칭성, 추이성을 만족시키고,  
`asPoint()` 메서드를 통해 `Point`로 변환 가능하므로 필요한 경우 위치만 따로 비교가 가능하다.

자바 라이브러리에도 구체 클래스를 extends 하여 값을 추가한 클래스가 종종 있다.  
`java.sql.Timestamp`는 `java.util.Date`를 확장한 후 nanoseconds 필드를 추가했다.

그 결과로 `Timestamp`의 `equals`는 대칭성을 위배하며, `Date` 객체와 한 컬렉션에 넣거나 섞어 사용하면 엉뚱하게 동작할 수 있다.

`Timestamp`를 이렇게 설계한 것은 실수이니 절대 따라 해서는 안된다.

### 일관성

일관성은 두 객체가 같다면 앞으로도 영원히 같아야 한다는 뜻이다.  
(어느 하나 혹은 두 객체 모두가 수정되지 않는 한)

클래스가 가변이든 불변이든 `equals`의 판단에 신뢰할 수 없는 자원이 끼어들게 해서는 안 된다.

예를 들어 `java.net.URL`의 `equals`는 주어진 URL과 매핑된 호스트의 IP 주소를 이용해 비교한다.  
호스트 이름을 IP 주소로 바꾸려면 네트워크를 통해야 하는데, 그 결과가 항상 같다고 보장할 수 없다.

```java
URL u1 = new URL("http://example.com");
URL u2 = new URL("http://93.184.216.34"); // example.com의 실제 IP 주소
System.out.println(u1.equals(u2));  // true
```

하지만 CDN을 사용하거나 DNS 라운드로빈 등  
DNS 조회 결과가 시간이나 네트워크 상태에 따라 다를 수 있기 때문에

URL의 `equals`는 일관성을 어긴다.

이런 문제를 피하려면 `equals`는 항시 메모리에 존재하는 객체만을 사용한 결정적(deterministic) 계산만 수행해야 한다.

### null-아님

null-아님 은 모든 객체가 `null`과 같지 않아야 한다는 뜻이다.

객체와 `null`을 비교하는 경우에 `NullPointerException`을 던지는 경우도 규약 위반이다.

그렇다고 if(o == null) return false; 처럼 명시적으로 검사할 필요는 없는데, instanceof가 null에 대해 안전하기 때문에

```java
if (!(o instanceof MyType))
    return false;
```

이 코드에서 o가 `null` 이라면 instanceof는 자동으로 false를 반환하기 때문이다.

## 양질의 equals 구현 방법 총 정리

1. == 연산자를 이용해 입력이 자기 자신의 참조인지 확인한다.

   자기 자신이면 true를 반환하도록 한다.  
   이것은 단순 성능 최적화 용도인데, 비교 작업이 복잡한 상황일 때 효과가 좋다.

2. instancof 연산자로 입력이 올바른 타입인지 확인한다.

   그렇지 않다면 false를 반환한다. 이때의 올바른 타입은 `equals`가 정의된 클래스인 것이 보통이지만, 가끔은 그 클래스가 구현한 특정 인터페이스가 될 수도 있다.

   어떤 인터페이스는 자신을 구현한 서로 다른 클래스끼리도 비교할 수 있도록 `equals` 규약을 수정하기도 하는데, 이런 인터페이스를 구현한 클래스라면 `equals`에서 해당 인터페이스를 사용해야 한다.

   Set, List, Map, Map.Entry 등의 컬렉션 인터페이스 들이 여기에 해당한다.

   ```java
   import java.util.*;

   public class ListEqualsExample {
       public static void main(String[] args) {
           List<String> list1 = new ArrayList<>();
           list1.add("A");
           list1.add("B");

           List<String> list2 = new LinkedList<>();
           list2.add("A");
           list2.add("B");

           System.out.println(list1.equals(list2)); // true
           System.out.println(list2.equals(list1)); // true
       }
   }
   ```

   `List` 인터페이스 구현체들의 `equals`는 이렇게 되어 있다.

   ```java
   @Override
   public boolean equals(Object o) {

       // 입력 타입을 검사할 때 `List` 인터페이스를 기준으로 검사
       if (!(o instanceof List))
           return false;

       List<?> other = (List<?>) o;
       if (other.size() != this.size()) return false;

       Iterator<?> it1 = this.iterator();
       Iterator<?> it2 = other.iterator();
       while (it1.hasNext()) {
           if (!Objects.equals(it1.next(), it2.next()))
               return false;
       }
       return true;
   }
   ```

3. 입력을 올바른 타입으로 형변환한다.

   앞서 2번에서 instanceof 검사를 했기 때문에 이 단계는 100% 성공한다.

4. 입력 객체와 자기 자신의 대응되는 '핵심' 필드들이 모두 일치하는지 하나씩 검사한다.

   모든 필드가 일치하면 true, 하나라도 다르면 false를 반환  
   2단계에서 인터페이스를 사용했다면 입력의 필드 값을 가져올 때도 그 인터페이스의 메서드를 사용해야 한다.

---

float과 double을 제외한 기본 타입 필드는 == 연산자로 비교하고, 참조 타입 필드는 각각의 equals로, float과 double 필드는 각각 정적 메서드인 `Float.compare(float, float)`과 `Double.compare(double, double)`로 비교한다.(부동소수점)

`Float.equals`와 `Double.equals` 메서드를 대신 사용할 수도 있지만, 이 메서드들은 오토박싱을 수반할 수 있어 성능상 좋지 않다.

때로는 `Null`도 정상 값으로 취급하는 참조 타입 필드가 있을 수 있는데, 이런 필드는 정적 메서드인 `Object.equals(Object, Object)`로 비교해 `NullPointerException` 발생을 예방하자.

앞서의 `CaseInsensitiveString` 처럼 비교하기가 아주 복잡한 필드를 가진 클래스도 있을 수 있다.  
이럴 때는 그 필드의 표준형(Canonical Form)을 저장해둔 후 표준형끼리 비교하면 훨씬 경제적이다.  
이 기법은 특히 불변 클래스에 알맞다. 가변 객체라면 값이 바뀔 때마다 표준형도 갱신해줘야 한다.

어떤 필드를 먼저 비교하느냐가 `equals`의 성능을 좌우하기도 한다.  
성능을 높이려면 다를 가능성이 더 크거가 비교하는 비용이 싼 필드를 먼저 비교

동기화용 락 필드 같이 객체의 논리적 상태와 관련 없는 필드는 비교하면 안된다.

핵심 필드로부터 계산해낼 수 있는 파생 필드 역시 굳이 비교할 필요는 없지만, 파생 필드를 비교하는 쪽이 더 빠를 수도 있다.  
예를 들어 자신의 영역을 캐싱해두는 `Polygon` 클래스가 있다고 했을 때, 각 변과 정점을 일일이 비교할 필요 없이 캐시해둔 영역만 비교하면 결과를 곧바로 알 수 있다.

`equals`를 다 구현했다면 대칭적인지, 추이성이 있는지, 일관성이 있는지 꼭 확인하자.  
단위 테스트를 작성해 돌려보자.  
반사성과 null-아님 도 만족해야 하지만, 이 둘이 문제되는 경우는 별로 없다.

## 진짜 마지막 주의사항

- `equals`를 재정의할 땐 `hashCode`도 반드시 재정의하자
- 너무 복잡하게 해결하려 들지 말자

  필드들의 동치성만 검사해도 equals 규약을 어렵지 않게 지킬 수 있다.

  일반적으로 별칭(alias)은 비교하지 않는게 좋다.  
  예를 들어 `File` 클래스라면 심볼릭 링크를 비교해 같은 파일을 가리키는지를 확인하려 들면 안된다.

  ```java
    File f1 = new File("/home/user/docs/file.txt");
    File f2 = new File("/home/user/docs/shortcut"); // 실제로는 file.txt에 대한 심볼릭 링크
    f1.equals(f2); // 심볼릭 링크를 따라가서 같은 파일이면 true?
  ```

  복잡하고, equals 규약 지키기도 어렵다.

* `Object` 외의 타입을 매개변수로 받는 `equals` 메서드는 선언하지 말자

  ```java
  // Broken - parameter type must be Object!
  public boolean equals(MyClass o) {
  ...
  }
  ```

  이 메서드는 `Object.equals`를 재정의한 것이 아니라 다중정의한 것이다.  
  이 메서드는 `equals(Object)`를 제대로 오버라이딩 한 것이 아닌데도 `@Override`를 붙이면 마치 오버라이딩 한 것 처럼 개발자가 착각하도록 만들고(Java 5에서는 `equals(MyClass o) 에 `@Override`를 붙여도 컴파일 에러가 나지 않았음), 보안 측면에서도 잘못된 정보를 준다.

---

`equals`, `hashCode`를 작성하고 테스트하는 일은 지루하고, 이를 테스트하는 코드도 항상 뻔하다.

구글이 만든 AutoValue 프레임워크를 사용하면 이 작업을 대신해준다!!

클래스에 어노테이션 하나만 추가하면 AutoValue가 이 메서드들을 알아서 작성해준다.
