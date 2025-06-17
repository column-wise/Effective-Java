# item 24. 멤버 클래스는 되도록 static으로 만들라

nested class란 다른 클래스 안에 정의된 클래스를 말한다.

nested 클래스는 자신을 감싼 바깥 클래스에만 쓰여야하며, 그 외 쓰임새가 있다면 톱레벨 클래스로 빼야 한다.

중첩 클래스의 종류는

1. 정적 멤버 클래스
2. (비정적) 멤버 클래스
3. 익명 클래스
4. 지역 클래스

## 정적 멤버 클래스

바깥 클래스의 private 멤버에도 접근할 수 있다.?? static이 아니어도 가능한디

| 항목                               | static nested class       | inner class (non-static)   |
| ---------------------------------- | ------------------------- | -------------------------- |
| outer 클래스의 `private` 멤버 접근 | ✅ 가능                   | ✅ 가능                    |
| outer 클래스의 인스턴스 멤버 접근  | ❌ 불가능 (인스턴스 없음) | ✅ 가능 (암시적 참조 존재) |
| outer 인스턴스 필요 여부           | ❌ 필요 없음              | ✅ 필요함                  |

주로 바깥 클래스와 함께 쓰일 때만 유용한 public 도우미 클래스로 쓰인다.

계산기가 지원하는 연산 종류를 정의하는 enum 타입을 생각해보면, Operation enum 타입은 Calculator 클래스의 public 정적 멤버 클래스가 되어야 한다.

```java
public class Calculator {

    // static 멤버 enum
    public static enum Operation {
        PLUS {
            @Override
            public double apply(double x, double y) {
                return x + y;
            }
        },
        MINUS {
            @Override
            public double apply(double x, double y) {
                return x - y;
            }
        },
        MULTIPLY {
            @Override
            public double apply(double x, double y) {
                return x * y;
            }
        },
        DIVIDE {
            @Override
            public double apply(double x, double y) {
                return x / y;
            }
        };

        public abstract double apply(double x, double y);
    }

    // Calculator 클래스의 다른 기능들...
}
```

사용 예시

```java
public class Main {
    public static void main(String[] args) {
        double x = 4.0;
        double y = 2.0;

        for (Calculator.Operation op : Calculator.Operation.values()) {
            System.out.printf("%f %s %f = %f%n",
                x, op.name(), y, op.apply(x, y));
        }
    }
}
```

## 비정적 멤버 클래스

static이 없는 nested class

비정적 멤버 클래스의 인스턴스는 바깥 클랫스의 인스턴스와 암묵적으로 연결된다.  
그래서 비정적 멤버 클래스의 인스턴스 메서드에서 정규화된 this를 사용해 바깥 인스턴스의 메서드를 호출하거나 바깥 인스턴스의 참조를 가져올 수 있다.

### inner class 자신 참조(this)

```java
public class Outer {
    private String name = "Outer";

    public class Inner {
        private String name = "Inner";

        public void print() {
            System.out.println(this.name); // ✅ Inner의 name
        }
    }
}
```

### inner class에서 outer class 참조(정규화된 this/`OuterClassName.this`)

```java
public class Outer {
    private String name = "Outer";

    public class Inner {
        private String name = "Inner";

        public void print() {
            System.out.println(this.name);        // Inner의 name
            System.out.println(Outer.this.name);  // Outer의 name
        }
    }
}
```

inner class의 인스턴스가 outer class의 인스턴스와 독립적으로 존재해야 한다면
static으로 만들어야 한다. non-static 멤버 클래스는 바깥 인스턴스 없이는 생성할 수 없기 때문이다.

non-static inner class의 인스턴스와 바깥 인스턴스의 관계는 inner class가 인스턴스화될 때 확립되며, 더 이상 변경할 수 없다.

또한 static inner class는 outer 인스턴스에 대한 참조를 가지지 않는다.  
(non-static은 inner class 생성자 호출시에 컴파일러가 참조를 넣어줌)

---

비정적 멤버 클래스는 어댑터를 정의할 때 자주 쓰인다.

```java
public class MySet<E> extends AbstractSet<E> {

    private E[] elements;

    @Override
    public Iterator<E> iterator() {
        return new MyIterator();
    }

    private class MyIterator implements Iterator<E> {
        private int cursor = 0;

        @Override
        public boolean hasNext() {
            return cursor < elements.length;
        }

        @Override
        public E next() {
            return elements[cursor++];
        }
    }
}
```

AbstractSet을 implement 해서 MySet을 구현하려고 함

MySet은 내부 데이터구조 elements[]를 가지고 있는데,  
Override한 iterator에서는 이 데이터구조의 형태를 모름

MyIterator는 MySet의 내부구조를 알기 때문에 Iterator 인터페이스를 구현하면서 내부 데이터와 클라이언트 요구를 이어주는 역할을 할 수 있음(Adapter)

Target: Iterator<E> 인터페이스  
Adaptee: MySet 내부데이터 elements  
Adapter: MyIterator 클래스

inner class가 바깥 인스턴스(elements)에 접근해야 하기 때문에 MyIterator를 static으로 만들면 접근할 수 없음

접근할 일이 없다면 static을 붙여서 정적 멤버 클래스로 만들자.

## 익명 클래스

이름이 없다..

익명클래스는 바깥 클래스의 멤버도 아니다.

쓰이는 시점에 선언과 동시에 인스턴스가 만들어진다.

상수 변수 이외의 정적 멤버는 가질 수 없다.

```java
import java.util.*;

public class AnonymousExample {
    public static void main(String[] args) {
        // 1. Runnable 인터페이스를 구현하는 익명 클래스
        Runnable task = new Runnable() {
            @Override
            public void run() {
                System.out.println("Hello from anonymous class!");
            }
        };

        task.run();  // 실행

        // 2. Comparator를 익명 클래스로 구현
        List<String> names = Arrays.asList("Charlie", "Alice", "Bob");

        Collections.sort(names, new Comparator<String>() {
            @Override
            public int compare(String a, String b) {
                return b.compareTo(a); // 역순 정렬
            }
        });

        System.out.println(names); // [Charlie, Bob, Alice]
    }
}
```

> 오직 비정적인 문맥에서 사용될 때만 바깥 클래스의 인스턴스를 참조할 수 있다. 정적 문맥에서라도 상수 변수 이외의 정적 멤버는 가질 수 없다. 즉 상수 표현을 위해 초기화된 final 기본 타입과 문자열 필드만 가질 수 있다.

?? 뭔 말이야

| 구분                                   | 의미                                                                     | 예시                                                           |
| -------------------------------------- | ------------------------------------------------------------------------ | -------------------------------------------------------------- |
| **정적인 문맥 (static context)**       | **인스턴스 없이 실행 가능한 영역**<br>즉, `static` 키워드가 붙은 곳      | `static` 메서드, static 블록, static 필드 초기화               |
| **비정적인 문맥 (non-static context)** | **인스턴스가 있어야 실행 가능한 영역**<br>즉, 인스턴스 메서드, 생성자 등 | `main()` 외 인스턴스 메서드, 생성자 내부, 인스턴스 초기화 블록 |

#### 익명 클래스가 비정적 문맥에서 선언

```java
public class Outer {
    private int outerValue = 42;

    public void doSomething() {
        Runnable r = new Runnable() {
            @Override
            public void run() {
                System.out.println(outerValue); // ✅ 접근 가능!
            }
        };
        r.run();
    }
}
```

doSomeThing이 static이 아니므로 외부 클래스의 인스턴스에 대한 참조가 생기고, 접근이 가능한 것

#### 익명 클래스가 정적 문맥에서 선언

```java
public class Outer {
    private int outerValue = 42;

    public static void main(String[] args) {
        Runnable r = new Runnable() {
            @Override
            public void run() {
                // System.out.println(outerValue); // ❌ 컴파일 에러
                System.out.println("Hello");
            }
        };
        r.run();
    }
}
```

main 함수가 static 이기 때문에 Outer.this 같은 참조가 생기지 않아 outerValue에 접근이 불가하다.

## 지역 클래스

지역 클래스는 지역변수를 선언할 수 있는 곳이면 어디든 선언할 수 있다.

멤버 클래스처럼 이름이 있고, 반복해서 사용할 수 있다.

익명 클래스처럼 비정적 문맥에서 사용될 때만 바깥 인스턴스를 참조할 수 있으며,  
정적 멤버는 가질 수 없다.

```java
public class LocalClassExample {
    private int outerValue = 10;

    public void doSomething() {
        int localVar = 5; // effectively final(final이 붙지 않았지만 컴파일러가 보기엔 final. 수정이 안되니까)

        // 지역 클래스 선언: 메서드 내부에서 선언됨
        class LocalPrinter {
            public void print() {
                System.out.println("outerValue = " + outerValue); // ✅ outer 인스턴스 참조 가능
                System.out.println("localVar = " + localVar);     // ✅ 지역 변수 접근 (final 또는 effectively final일 때만)
            }
        }

        LocalPrinter printer = new LocalPrinter();
        printer.print();
    }

    public static void main(String[] args) {
        new LocalClassExample().doSomething();
    }
}
```
