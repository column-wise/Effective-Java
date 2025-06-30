# item 42. 익명 클래스보다는 람다를 사용하라

예전에는 자바에서 함수 타입을 표현할 때 추상 메서들르 하나만 담은 인터페이스(드물게는 추상 클래스)를 사용했다.  
이런 인터페이스의 인스턴스를 함수 객체(function object)라고 하여, 특정 함수나 동작을 나타내는 데 썼다.

## 익명 클래스

ex.

```java
interface IntBinaryOperation {
    int apply(int a, int b);
}

public class Calculator {
    public static void main(String[] args) {
        IntBinaryOperation add = new IntBinaryOperation() {
            @Override
            public int apply(int a, int b) {
                return a + b;
            }
        };

        // 익명 내부 클래스(anonymous inner class)로 일회용 구현체 생성
        IntBinaryOperation multiply = new IntBinaryOperation() {
            @Override
            public int apply(int a, int b) {
                return a * b;
            }
        };

        System.out.println("10 + 5 = " + operate(10, 5, add));
        System.out.println("10 * 5 = " + operate(10, 5, multiply));
    }

    public static int operate(int a, int b, IntBinaryOperation op) {
        return op.apply(a, b);
    }
}
```

책에 있는 예시

```java
interface Comparator<T> {
    int compare(T o1, T o2); // 추상 메서드 1개 → 함수형 인터페이스
}
```

```java
Collections.sort(words, new Comparator<String>() {
    @Override
    public int compare(String s1, String s2) {
        return Integer.compare(s1.length(), s2.length());
    }
});
```

## 람다식

과거에는 함수 객체를 나타내는 것에 익명 클래스면 충분했다.  
하지만 익명 클래스 방식은 코드가 너무 길기 때문에 자바는 함수형 프로그래밍에 적합하지 않았다.

자바 8에 와서 추상 메서드 하나짜리 인터페이스(함수형 인터페이스)의 인스턴스를 람다식(lambda expression)을 이용해 만들 수 있게 되었다.

```java
// Lambda expression as function object (replaces anonymous class)
Collections.sort(words,
    (s1, s2) -> Integer.compare(s1.length(), s2.length()));
```

매개변수는 String, 리턴 타입은 Integer인데, 이건은 컴파일러가 문맥을 살펴 추론해준다.  
상황에 따라 컴파일러가 타입을 결정하지 못할 수도 있는데, 그럴 때는 프로그래머가 직접 명시해야 한다.

람다 자리에 비교자 생성 메서드를 사용하면 이 코드를 더 간결하게 만들 수도 있다.  
(자바 8에 추가된 Comparator의 정적 팩토리 메서드)

```java
Collection.sort(words, comparingInt(String::length));
```

더 나아가 자바 8 때 List 인터페이스에 추가된 sort 메서드를 이용하면 더 짧아진다.

```java
words.sort(comparingInt(String::length));
```

### ※ 메서드 참조(Method Reference) "::"

```java
String::length
```

는

```java
(String s) -> s.length()
```

같은 의미를 가진다.

---

item 34에서의 Operation Enum 타입 예시를 보자.

```java
// Enum type with constant-specific class bodies & data (Item 34)
// 컴파일 타임에 자동으로 각 enum 상수(PLUS, MINUS, TIMES, DIVIDE)에 대한 유일한 인스턴스가 생성됨
    public enum Operation {
    PLUS("+") {
        public double apply(double x, double y) { return x + y; }
    },
    MINUS("-") {
        public double apply(double x, double y) { return x - y; }
    },
    TIMES("*") {
        public double apply(double x, double y) { return x * y; }
    },
    DIVIDE("/") {
       public double apply(double x, double y) { return x / y; }
    };

    private final String symbol;
    Operation(String symbol) { this.symbol = symbol; }
    @Override public String toString() { return symbol; }
    public abstract double apply(double x, double y);
}
```

람다를 이용하면 열거 타입의 인스턴스 필드를 이용하는 방식으로 상수별로 다르게 동작하는 코드를 쉽게 구현할 수 있다.

```java
// Enum with function object fields & constant-specific behavior
public enum Operation {
    PLUS ("+", (x, y) -> x + y),
    MINUS ("-", (x, y) -> x - y),
    TIMES ("*", (x, y) -> x * y),
    DIVIDE("/", (x, y) -> x / y);

    private final String symbol;
    private final DoubleBinaryOperator op;

    Operation(String symbol, DoubleBinaryOperator op) {
        this.symbol = symbol;
        this.op = op;
    }

    @Override public String toString() { return symbol; }
    public double apply(double x, double y) {
        return op.applyAsDouble(x, y);
    }
}
```

오 훨씬 깔끔하다  
그렇다면 람다를 이용하면 모든 상수별 몸체

```java
MINUS("-") {
        public double apply(double x, double y) { return x - y; }
    }
```

이런거 전부 대체할 수 있는 것이 아닌가??

-> 그렇지 않다.

메서드와 클래스와는 달리 람다는 이름이 없고 문서화할 수도 없다.  
따라서 코드 자체로 동작이 설명되지 않거나 코드 줄 수가 많아지면 람다를 쓰지 말아야 한다.

```java
// 나쁜 람다 사용 예
PLUS("+", (x, y) -> {
    // 5줄 이상 길고 복잡한 연산
    log.info("start");
    ...
    return complicatedResult;
})
```

이럴 땐 상수별 클래스 몸체를 쓰자

```java
PLUS("+") {
    @Override
    public double apply(double x, double y) {
        // 길고 복잡한 연산
        if (x < 0 || y < 0) {
            throw new IllegalArgumentException("...");
        }
        return Math.pow(x + y, 2);
    }
}
```

1. 람다가 길거나 읽기 어렵다면 리팩토링하거나 람다를 쓰지 않는 쪽으로 고쳐야 한다.
2. 또한 enum 생성자에 넘기는 람다는 static context에서 실행되므로 enum 인스턴스의 필드나 메서드에 접근할 수 없다.

ex.

```java
SOME("value", () -> this.doSomething()); // ❌ 컴파일 에러
```

## 람다의 한계

1. 함수형 인터페이스에서만 사용 가능
   람다는 추상 메서드가 하나만 있는 인터페이스(함수형 인터페이스)에서만 사용할 수 있다.

   → 추상 메서드가 여러 개인 인터페이스는 람다로 구현할 수 없다.  
    이 경우 익명 클래스를 사용해야 한다.

2. 추상 클래스는 람다로 구현 불가  
   람다는 인터페이스에만 쓸 수 있다.

   따라서 추상 클래스의 인스턴스를 만들 땐 반드시 익명 클래스를 사용해야 한다.

3. 람다에서는 this가 바깥을 가리킨다  
   람다 내부에서 this는 바깥 클래스 인스턴스를 가리킨다.

   반면, 익명 클래스에서는 this가 그 익명 클래스 자신의 인스턴스를 가리킨다.

   → 함수 객체가 자기 자신을 참조해야 하는 경우, 반드시 익명 클래스를 써야 한다.

4. 직렬화는 피하자  
   람다 역시 익명 클래스처럼 직렬화 형태가 구현마다 다르다.

   따라서 람다를 직렬화해서 저장하거나 전송하는 건 매우 위험하다.  
    → 되도록 직렬화를 피할 것.
