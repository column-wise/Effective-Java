# item 29. 이왕이면 제네릭 타입으로 만들라

## ex. [item 7](https://github.com/column-wise/Effective-Java/blob/main/Chap.02/item_07.md)에서 다룬 단순한 스택 코드를 제네릭을 이용하도록 바꿔보자

```java
// Object-based collection - a prime candidate for generics
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        return result;
    }

    public boolean isEmpty() {
        return size == 0;
    }

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

제네릭을 사용하도록 바꿔보자.

일반 클래스를 제네릭 클래스로 만드는 첫 단계는 클래스 선언에 타입 매개 변수를 추가하는 일이다.  
(public class Stack -> public class Stack<E>)

그리고 스택이 담을 원소의 타입 하나만 추가해보자  
(Object[] elements -> E[] elements)

```java
// Initial attempt to generify Stack - won't compile!
public class Stack<E> {
    private E[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new E[DEFAULT_INITIAL_CAPACITY]; // ❌ 컴파일 에러 발생
    }

    public void push(E e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public E pop() {
        if (size == 0)
            throw new EmptyStackException();
        E result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        return result;
    }

    // ... no changes in isEmpty or ensureCapacity
}
```

컴파일이 안된다

```
Stack.java:8: generic array creation
    elements = new E[DEFAULT_INITIAL_CAPACITY];
                ^
```

E가 실체화 불가라 배열을 만들 수 없다

### 제네릭 배열 생성 불가 우회법

1. Object 배열을 생성한 다음 제네릭 배열로 형 변환

   ```java
   elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
   ```

   경고 발생

   ```
   Stack.java:8: warning: [unchecked] unchecked cast
   found: Object[], required: E[]
       elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
                   ^
   ```

   컴파일러가 보기에 타입 안전하지 않다는 뜻  
   하지만 배열 elements는 private이고, 클라이언트에 반환되거나 다른 메서드에 전달될 일이 없으므로 원소는 항상 E가 담긴다.

   그래서 `@SuppressWarnings` 어노테이션을 사용해 경고를 숨겨도 될 것 같다

   ```java
   // 배열 elements는 push(E)로 넘어온 E 인스턴스만 담는다.
   // 따라서 타입 안전성을 보장하지만,
   // 이 배열의 런타임 타입은 E[]가 아닌 Object[] 다
   @SuppressWarnings("unchecked")
    public class Stack<E> {
        private E[] elements;

        @SuppressWarnings("unchecked")
        public Stack() {
            elements = (E[]) new Object[10]; // 강제로 E[]처럼 취급
        }

        public void push(E e) { elements[size++] = e; }
        public E pop() { return elements[--size]; }
    }

   ```

2. elements 필드의 타입을 E[]에서 Object[]로 다시 바꾼다

   private E[] elements; -> private Object[] elements

   오류가 난다..

   ```
   Stack.java:19: incompatible types
   found: Object, required: E
       E result = elements[--size];
                          ^
   ```

   elements 원소가 Object라서 E에 담을 수 없다

   다시 형변환을 하면???
   경고가 뜬다..

   ```
   Stack.java:19: warning: [unchecked] unchecked cast
    found: Object, required: E
        E result = (E) elements[--size];
                                ^
   ```

   E가 실체화 불가라 컴파일러는 런타임에 이뤄지는 형변환이 안전한지 알 수 없다  
   하지만 역시 elements가 private이고, 외부로 전달되지 않으니까 경고를 숨겨도 될 것 같다

   ```java
   public class Stack<E> {
       private Object[] elements;

       public Stack() {
           elements = new Object[10];
       }

       public void push(E e) {
           elements[size++] = e;
       }

       @SuppressWarnings("unchecked")
       public E pop() {
           return (E) elements[--size]; // 꺼낼 때 캐스팅
       }
   }
   ```

   첫 번째 방식은 형 변환을 배열 생성 시 단 한 번만 해주면 되지만  
   두 번째 방식에서는 배열에서 원소를 읽을 때마다 해줘야 하므로 첫 번째 방식이 더 선호된다

   하지만 배열의 런타임 타입이 컴파일타임 타입과 일치하지 않는 상태인 힙 오염(heap pollution)을 일으킨다...

   스택 예시에서는 배열이 스택 내부에서만 쓰였고, Push(E) 로만 들어가므로 안전했음

## 제네릭으로 바꾼 스택 사용해보기

```java
// Little program to exercise our generic Stack
public static void main(String[] args) {
 Stack<String> stack = new Stack<>();
 for (String arg : args)
     stack.push(arg);
 while (!stack.isEmpty())
     System.out.println(stack.pop().toUpperCase());
}
```

Stack에서 꺼낸 원소에서 String의 toUpperCase 메서드를 호출할 때 명시적 형변환을 수행하지 않으며, 컴파일러에 의해 자동 생성된 형변환이 항상 성공
