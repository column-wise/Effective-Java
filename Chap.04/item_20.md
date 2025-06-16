# item 20. 추상 클래스보다는 인터페이스를 우선하라

## 인터페이스의 장점

1.  인터페이스는 다중 구현이 가능하지만 추상 클래스는 단일 상속만 가능하다

2.  기존 클래스에 쉽게 인터페이스를 덧붙일 수 있음

    기존 클래스에 추상 클래스를 끼워넣기는 어렵다.  
    두 클래스가 같은 추상 클래스를 확장하길 원한다면,  
    그 추상 클래스는 계층 구조상 두 클래스의 공통 조상이어야 한다.

    ex. `Dog`와 `Printer`가 같은 추상 클래스를 상속받길 원함

    `Loggable` 이라는 추상 클래스를 상속받아서 `log(String msg)` 메서드를 공통으로 사용하고 싶다....

    ```java
    public class Dog extends Animal {
    ...
    }

    public class Printer extends Device {
        ...
    }
    ```

    ```java
    public abstract class Loggable {
         protected void log(String msg) {
             System.out.println("[LOG] " + msg);
         }
     }

     public class Dog extends Loggable { ... }      // ❌ 이미 Animal을 상속 중
     public class Printer extends Loggable { ... }  // ❌ 이미 Device를 상속 중

    ```

    그럼 Animal이랑 Device가 Loggable을 상속받도록 하면 안되냐??  
    -> 아. 그건 쉽지 않음

    Animal과 Device가 이미 존재하는 외부 라이브러리 또는 JDK 클래스일 수 있기 때문

    `Loggable`을 인터페이스로 구현하면 문제 없음

    ```java
    public interface Loggable {
         default void log(String msg) {
             System.out.println("[LOG] " + msg);
         }
     }

     public class Dog extends Animal implements Loggable { ... }
     public class Printer extends Device implements Loggable { ... }
    ```

3.  믹스인(Mixin) 구현에 적합하다

    믹스인이란 객체 지향 프로그래밍에서 기능 조각을 다른 클래스에 혼합해서 부여할 수 있는 설계 패턴 또는 개념이다.

    추상 클래스로는 믹스인을 정의할 수 없다..기존 클래스에 덧씌울 수 없기 때문

    클래스는 두 부모를 가질 수 없고, 클래스 계층 구조에는 믹스인을 삽입하기에 합리적인 위치가 없다.

4.  비계층적(non-hierarchical) 타입 설계 가능
    타입을 계층적으로 정의하면 수많은 개념을 구조적으로 잘 표현할 수 있지만, 현실에서는 계층을 엄격히 구분하기 어려운 개념이 있다.

    ex. 싱송라

    ```java
    public interface Singer {
        AudioClip sing(Song s);
    }
    ```

    ```java
    public interface Songwriter {
         Song compose(int chartPosition);
     }
    ```

    ```java
    public interface SingerSongwriter extends Singer, Songwriter {
        AudioClip strum();
        void actSensitive();
    }
    ```

5.  인터페이스는 기능을 향상시키는 안전하고 강력한 수단이다.

    타입을 추상클래스로 정의하면 그 타입에 기능을 추가하는 수단은 상속 뿐이다.

    상속해서 만든 클래스는 래퍼 클래스보다 활용도가 떨어지고 깨지기는 더 쉽다.

6.  default 메서드를 활용하면 인터페이스도 구현을 제공할 수 있다.

## 인터페이스가 갖는 제약

- equals hashCode, toString 같은 Object의 메서드는 default 메서드로 제공해서는 안된다.

  1. 다이아몬드 문제 발생

     클래스가 여러 인터페이스를 동시에 구현할 때, 각 인터페이스가 서로 다른 equals/hashCode/toString 기본 구현을 제공하면 충돌 발생

  2. 객체 정체성과 계약(Contract) 보존

     equals와 hashCode는 동등성·해시 불변식, toString은 표현 규약을 지니는데, 이 규약은 구현체의 내부 상태에 깊이 의존한다.

     외부 인터페이스가 default로 임의의 일반 구현을 주입하면 실제 클래스가 지녀야하는 불변식이 깨질 수 있다.

- 인스턴스 필드를 가질 수 없다.
- 내가 만들지 않은 인터페이스에는 default 메서드를 추가할 수 없다.

## 제약 해결: 인터페이스와 skeletal implementation(추상 골격 구현)

인터페이스는 타입을 정의하고, 스켈레톤 클래스(추상 클래스)는 핵심 구현을 제공
**템플릿 메서드 패턴(Template Method Pattern)**

ex.  
인터페이스 List<E> 를 구현하는 AbstractList<E> (skeleton implementation)를 이용하여 get, set, size를 제공하는 커스텀 리스트를 만드는 예시

```java
// Concrete implementation built atop skeletal implementation
static List<Integer> intArrayAsList(int[] a) {
    Objects.requireNonNull(a);

    // Java 9 이상에서는 <> (다이아몬드 연산자) 사용 가능
    // Java 8 이하는 <Integer> 명시 필요
    return new AbstractList<>() {
        @Override
        public Integer get(int i) {
            return a[i]; // Autoboxing (int → Integer)
        }

        @Override
        public Integer set(int i, Integer val) {
            int oldVal = a[i];
            a[i] = val; // Auto-unboxing (Integer → int)
            return oldVal; // Autoboxing
        }

        @Override
        public int size() {
            return a.length;
        }
    };
}
```

AbstractList 구현이 없다면

```java
    static List<Integer> intArrayAsList(int[] a) {
    Objects.requireNonNull(a);

    return new List<Integer>() {
        @Override
        public int size() {
            return a.length;
        }

        @Override
        public Integer get(int i) {
            return a[i]; // Autoboxing
        }

        @Override
        public Integer set(int i, Integer val) {
            int oldVal = a[i];
            a[i] = val;
            return oldVal;
        }

        // 나머지 모든 List 메서드를 직접 구현해야 함 (엄청 많음!)
        @Override public boolean isEmpty() { return size() == 0; }
        @Override public boolean contains(Object o) { throw new UnsupportedOperationException(); }
        @Override public Iterator<Integer> iterator() { throw new UnsupportedOperationException(); }
        @Override public Object[] toArray() { throw new UnsupportedOperationException(); }
        @Override public <T> T[] toArray(T[] a) { throw new UnsupportedOperationException(); }
        @Override public boolean add(Integer e) { throw new UnsupportedOperationException(); }
        @Override public boolean remove(Object o) { throw new UnsupportedOperationException(); }
        @Override public boolean containsAll(Collection<?> c) { throw new UnsupportedOperationException(); }
        @Override public boolean addAll(Collection<? extends Integer> c) { throw new UnsupportedOperationException(); }
        @Override public boolean addAll(int index, Collection<? extends Integer> c) { throw new UnsupportedOperationException(); }
        @Override public boolean removeAll(Collection<?> c) { throw new UnsupportedOperationException(); }
        @Override public boolean retainAll(Collection<?> c) { throw new UnsupportedOperationException(); }
        @Override public void clear() { throw new UnsupportedOperationException(); }
        @Override public void add(int index, Integer element) { throw new UnsupportedOperationException(); }
        @Override public Integer remove(int index) { throw new UnsupportedOperationException(); }
        @Override public int indexOf(Object o) { throw new UnsupportedOperationException(); }
        @Override public int lastIndexOf(Object o) { throw new UnsupportedOperationException(); }
        @Override public ListIterator<Integer> listIterator() { throw new UnsupportedOperationException(); }
        @Override public ListIterator<Integer> listIterator(int index) { throw new UnsupportedOperationException(); }
        @Override public List<Integer> subList(int fromIndex, int toIndex) { throw new UnsupportedOperationException(); }
    };
}
```

추상 골격 클래스가 없으면 List 인터페이스 전체를 직접 구현해야 함

### 골격 구현의 장점

추상 클래스가 갖는 상속 제약을 피할 수 있음.
상속을 강제하지 않으면서도 재사용성 제공

일반적인 추상 클래스:

```java
// 단순 추상 클래스: 재사용 시 반드시 상속받아야 함
abstract class AbstractOperation {
    abstract void operate();
}
```

이것을 재사용하기 위해서는 반드시 `extends AbstractOperation`  
이미 다른 클래스를 상속하고 있다면 사용이 불가

골격 구현 클래스:

```java
// 1) 인터페이스 정의: 핵심 메서드 + default 메서드
public interface CountingAction {
    void perform();                     // 핵심 작업
    default void report() {             // 기본 리포트 제공
        System.out.println("No report available");
    }
}

// 2) 골격 구현 클래스: 상태와 공통 로직 제공
public abstract class AbstractCountingAction implements CountingAction {
    private int count = 0;              // 상태 필드

    @Override
    public final void perform() {       // 공통 로직: 호출 횟수 추적 + 위임
        count++;
        doPerform();
    }

    protected abstract void doPerform(); // 서브클래스가 구현할 메서드

    @Override
    public void report() {              // 공통 리포트 로직
        System.out.println("Performed " + count + " times.");
    }
}

```

사용자가 선택 가능

1. 골격 구현 클래스 상속

   ```java
   class LoggingAction extends AbstractCountingAction {
        @Override
        protected void doPerform() {
            System.out.println("Logging something important...");
        }
   }
   ```

   상태 관리, 공통 로직을 하나도 다시 작성하지 않고 재사용

2. 인터페이스만 구현

   ```java
    class ManualCountingAction implements CountingAction {
        private int count = 0;

        @Override
        public void perform() {
            System.out.println("Manual action performed!");
            count++;
        }

        @Override
        public void report() {
            System.out.println("Performed " + count + " times.");
        }
    }
   ```

   다중 구현 가능, 상태 관리와 리포트를 직접 구현

3. 모의 다중 상속(Simulated Multiple Inheritance)
   이미 다른 클래스를 상속하고 있어서 `extends AbstractCountingAction` 이 불가능한 경우

   ```java
   // 이미 다른 클래스를 상속하고 있다고 가정
   class SpecialTask extends SomeOtherBase implements CountingAction {
       // 1) 스켈레톤 구현을 상속받는 private inner class
       private class Impl extends AbstractCountingAction {
           @Override
           protected void doPerform() {
               System.out.println("Special action logic");
           }
       }

       // 2) 실제 작업은 이 impl이 담당
       private final AbstractCountingAction impl = new Impl();

       // 3) 인터페이스 메서드를 모두 impl로 포워딩
       @Override
       public void perform() {
           impl.perform();
       }

       @Override
       public void report() {
           impl.report();
       }
   }
   ```

   약간 복잡하지만 단일 상속 제약을 피할 수 있고, interface가 제공하지 못하는 상태 관리까지 추상 골격 클래스로 가능함
