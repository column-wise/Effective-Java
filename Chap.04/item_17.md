# item 17. 변경 가능성을 최소화하라

불변 클래스: 인스턴스의 내부 값을 수정할 수 없는 클래스  
가변 클래스보다 설계하고 구현하고 사용하기가 쉽다.  
오류가 생길 여지도 적고 훨씬 안전하다

## 클래스를 불변으로 만드는 방법

- 객체의 상태를 변경하는 메서드를 제공하지 않는다.
- 클래스를 확장할 수 없도록 한다.
- 모든 필드를 final로 선언한다.
- 모든 필드를 private으로 선언한다.
- 자신 외에는 내부의 가변 컴포넌트에 접근할 수 없도록 한다.  
   클래스에 가변 객체를 참조하는 필드가 하나라도 있다면 클라이언트가 그 객체의 참조를 얻을 수 없도록 해야 한다.  
   생성자, 접근자, readObject 메서드 모두에서 방어적 복사

  ```java
  public final class Period {
       private final Date start;
       private final Date end;

       public Period(Date start, Date end) {
           this.start = new Date(start.getTime()); // 🛡️ 방어적 복사
           this.end = new Date(end.getTime());     // 🛡️ 방어적 복사
           if (this.start.after(this.end))
               throw new IllegalArgumentException(this.start + " after " + this.end);
       }

       public Date start() {
           return new Date(start.getTime()); // 🛡️ 반환 시 방어적 복사
       }

       public Date end() {
           return new Date(end.getTime());   // 🛡️ 반환 시 방어적 복사
       }
   }
  ```

## 예시(복소수 클래스)

```java
// Immutable complex number class
public final class Complex {
    private final double re;
    private final double im;

    public Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }

    public double realPart() {
        return re;
    }

    public double imaginaryPart() {
        return im;
    }

    public Complex plus(Complex c) {
        return new Complex(re + c.re, im + c.im);
    }

    public Complex minus(Complex c) {
        return new Complex(re - c.re, im - c.im);
    }

    public Complex times(Complex c) {
        return new Complex(
            re * c.re - im * c.im,
            re * c.im + im * c.re
        );
    }

    public Complex dividedBy(Complex c) {
        double tmp = c.re * c.re + c.im * c.im;
        return new Complex(
            (re * c.re + im * c.im) / tmp,
            (im * c.re - re * c.im) / tmp
        );
    }

    @Override
    public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof Complex))
            return false;
        Complex c = (Complex) o;
        // compare 대신 ==을 쓰지 않는 이유는 부동소수점 비교 때문 (책 63쪽 참고)
        return Double.compare(c.re, re) == 0 &&
               Double.compare(c.im, im) == 0;
    }

    @Override
    public int hashCode()

```

사칙연산 메서드들이 인스턴스 자신은 수정하지 않고 새로운 Complex 객체를 생성해 반환하고 있다.

이런 방식을 사용하면 코드에서 불변이 되는 영역의 비율이 높아져 장점이 생긴다.

## 불변 객체의 특징

- 불변 객체는 단순하다.(생성된 시점의 상태를 파괴될 때까지 간직한다)
- 불변 객체는 기본적으로 Thread-Safe하다.
- 불변 객체는 안심하고 공유할 수 있다.  
   -> 불변 객체는 인스턴스를 중복 생성하지 않게 해주는 정적 팩토리 메서드를 활용하여 재사용하면 좋다. 복사해봐야 원본과 같으므로 방어적 복사도 필요 없다.
- 불변 객체끼리는 내부 데이터를 공유할 수 있다.  
   ex. `BigInteger` 클래스는 값의 부호와 크기를 따로 표현하는데, `negate` 메서드는 크기가 같고 부호가 반대인 새 `BigInteger`를 생성한다. `negate`로 새로 만들어진 `BigInteger` 인스턴스는 원본 인스턴스의 크기 객체를 그대로 참조한다.
- 객체를 만들 때 다른 불변 객체들로 구성 요소를 사용하면 이점이 많다.

  ```java
  public final class Name {
      private final String first;
      private final String last;

      public Name(String first, String last) {
          this.first = first;
          this.last = last;
      }

      // getter, equals, hashCode 생략
  }

  public class User {
      private Name name; // Name이 불변이면 User가 다루기 쉬워짐
      private int age;

      public void updateName(Name newName) {
          this.name = newName; // 내부 값 통째로 교체만 하면 됨
      }
  }
  ```

  Name이 불변이기 때문에 User 클래스가 안전해지고 유지보수가 쉬워짐

  ```java
  Map<Name, String> userMap = new HashMap<>();
  userMap.put(new Name("Alice", "Kim"), "UserID123");
  ```

  만약 Name이 가변이라면

  ```java
  Name n = new Name("Alice", "Kim");
  userMap.put(n, "UserID123");

  n.setFirst("Hacker"); // 키의 hashCode, equals 바뀜!
  userMap.get(n); // 못 찾음 → Map 무결성 깨짐
  ```

  ❓이름 같은 사람 있으면 못 쓰는거 아님?

- 불변 객체는 그 자체로 실패 원자성을 제공한다.
- 값이 다르면 반드시 독립된 객체로 만들어야 한다.  
   ex. 100만 비트짜리 BigInteger에서 비트 하나를 수정해야 하는 상황
  BigInteger는 불변 객체이므로 비트 하나가 바뀐 다른 객체를 새로 만들어야 함
  비트 하나 바꾸는데 O(N)

  반면 BitSet은 가변 객체이고, 원하는 비트를 수정하는 메서드를 제공 O(1)

  원하는 객체를 완성하기까지의 단계가 많고, 그 중간 단계에서 만들어진 객체들이 모두 버려진다면 성능 문제가 더 불거진다.

  해결법

  1. 내부에서만 사용하는 비공개 가변 동반 클래스 사용

     BigInteger는 모듈러 지수 같은 다단계 연산속도를 높여주는 가변 동반 클래스 MutableBigInteger를 package-private으로 사용하고 있다.

     ```java
     public BigInteger modPow(BigInteger exponent, BigInteger m) {
         MutableBigInteger base = new MutableBigInteger(this);
         MutableBigInteger result = base.mutableModPow(exponent, m);
         return result.toBigInteger(1);
     }
     ```

  2. 공개된 가변 동반 클래스 제공

     String - StringBuilder

     ```java
     String s = "Hello";
     s += " World";
     ```

     JVM에서 실제로는 이렇게 동작함:

     ```java
     s = new StringBuilder()
     .append("Hello")
     .append(" World")
     .toString();
     ```

## 불변객체를 만드는 또 다른 설계 방법

클래스가 불변임을 보장하기 위해 final 클래스로 선언하여 상속을 막았다.

더 유연한 방법이 있는데, 모든 생성자를 private이나 package-private으로 선언하고, public 정적 팩토리 메서드를 제공하는 방법이다.

```java
// Immutable class with static factories instead of constructors
public class Complex {
    private final double re;
    private final double im;

    private Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }

    public static Complex valueOf(double re, double im) {
        return new Complex(re, im);
    }

    // ... Remainder unchanged
}
```

이렇게 하면 패키지 내부에서는 구현 클래스를 원하는 만큼 만들어 사용할 수 있다.

```java
// 외부에 공개된 타입
public abstract class Complex {
    public abstract double re();
    public abstract double im();

    public static Complex valueOf(double re, double im) {
        if (re == 0 && im == 0) return Zero.INSTANCE;
        return new DefaultComplex(re, im);
    }
}

// 내부 구현 클래스들 (package-private)
class DefaultComplex extends Complex { ... }
class Zero extends Complex { static final Zero INSTANCE = new Zero(); ... }
```

패키지 바깥에서 보면 이 불변 객체는 protected나 public 생성자가 없어 확장할 수 없으므로 final이나 다름없다.

## 정리

1. 클래스는 꼭 필요한 경우가 아니라면 불변이어야 한다.
2. 불변으로 만들 수 없는 클래스라도 변경할 수 있는 부분을 최소한으로 줄이자.
3. 다른 합당한 이유가 없다면 모든 필드는 private final 이어야 한다.
4. 생성자는 불변식 설정이 모두 완료된, 초기화가 완벽히 끝난 산태의 객체를 생성해야 한다.

## 불변 객체 === record?

record: 불변 객체를 만들기 위한 공식 문법적 지원(Syntactic Sugar)

### 기존 불변 객체 선언 방식

```java
public final class Name {
    private final String first;
    private final String last;

    public Name(String first, String last) {
        this.first = first;
        this.last = last;
    }

    public String first() { return first; }
    public String last() { return last; }

    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { ... }
    @Override public String toString() { ... }
}
```

### record 사용

```java
public record Name(String first, String last) {}
```

#### 주의

완전 불변은 아님

```java
public record User(String name, List<String> tags) {}
```

User 자체는 불변이지만, tags는 List 이므로 외부에서 수정이 가능함(얕은 불변성)

```java
List<String> t = new ArrayList<>();
User u = new User("Alice", t);
t.add("HACK"); // User 내부 상태도 바뀜!
```

해결법

```java
public record User(String name, List<String> tags) {
    public User {
        if (name == null || tags == null)
            throw new NullPointerException();

        this.name = name;
        this.tags = List.copyOf(tags); // ✅ 진짜 불변
    }
}
```

List.copyOf를 사용하면 불변 객체가 반환됨

수정 시도 시 `UnsupportedOperationException` 발생
