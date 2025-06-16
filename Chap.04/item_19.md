# item 19. 상속을 고려해 설계하고 문서화하라. 그러지 않았다면 상속을 금지하라.

item 18에서 상속을 염두에 두지 않은 '외부' 클래스를 상속할 때의 위험을 설명했다.

## 상속을 고려한 설계와 문서화

1. 상속용 클래스는 재정의할 수 있는 메서드들을 내부적으로 어떻게 이용하는지(자기 사용) 문서로 남겨야 한다.

   클래스의 API로 공개된 메서드에서 자신의 또 다른 메서드를 호출할 수 있다.  
   이 메서드가 재정의 가능 메서드라면 그 사실을 API 설명에 명시해야 한다.  
   어떤 순서로 호출하는지, 각각의 호출 결과가 이어지는 처리에 어떻게 영향을 주는지

   API 문서의 메서드 설명 끝에 종종 "Implementation Requirements"로 시작하는 절이 있는데, 그 메서드의 내부 동작 방식을 설명하는 곳이다.

   ```
   public boolean remove(Object o)

    Removes a single instance of the specified element from this collection, if it
    is present (optional operation). More formally, removes an element e such
    that Objects.equals(o, e), if this collection contains one or more such
    elements. Returns true if this collection contained the specified element (or
    equivalently, if this collection changed as a result of the call).

    Implementation Requirements: This implementation iterates over the collection
    looking for the specified element. If it finds the element, it removes
    the element from the collection using the iterator’s remove method. Note that
    this implementation throws an UnsupportedOperationException if the
    iterator returned by this collection’s iterator method does not implement
    the remove method and this collection contains the specified object.
   ```

   itorator 메서드를 재정의 하면 remove 메서드의 동작에 영향을 줄 것이다.

   클래스를 안전하게 상속할 수 있게 하기 위해 내부 구현 방식을 설명해야 한다...

   이것은 좋은 API 문서는 '어떻게'가 아닌 '무엇'을 하는지를 설명해야 한다는 격언과 대치됨  
   상속이 캡슐화를 해치기 때문이다!!!!!

2. 클래스의 내부 동작 과정 중간에 끼어들 수 있는 훅(hook)을 잘 선별하여 protected 메서드 형태로 공개해야 할 수도 있다.

   java.util.AbstractList의 removeRange 메서드를 보자.

   ```
   protected void removeRange(int fromIndex, int toIndex)
   Removes from this list all of the elements whose index is between
   fromIndex, inclusive, and toIndex, exclusive. Shifts any succeeding
   elements to the left (reduces their index). This call shortens the list by
   (toIndex - fromIndex) elements. (If toIndex == fromIndex, this operation
   has no effect.)

   This method is called by the clear operation on this list and its sublists.
   Overriding this method to take advantage of the internals of the list implementation
   can substantially improve the performance of the clear operation
   on this list and its sublists.

   Implementation Requirements: This implementation gets a list iterator
   positioned before fromIndex and repeatedly calls ListIterator.next
   followed by ListIterator.remove, until the entire range has been
   removed. Note: If ListIterator.remove requires linear time, this
   implementation requires quadratic time.

   Parameters:
   fromIndex index of first element to be removed.
   toIndex index after last element to be removed.
   ```

   List 구현체의 최종 사용자는 removeRange 메서드에 관심이 없다.  
   이것은 List를 상속해서 직접 구현하는 개발자를 위한 메서드이고, `clear` 메서드는 내부적으로 `removeRange`를 호출함.

   ```java
   @Override
    public void clear() {
        removeRange(0, size());
    }
   ```

   ```java
   protected void removeRange(int fromIndex, int toIndex) {
        ListIterator<E> it = listIterator(fromIndex);
        for (int i = 0; i < toIndex - fromIndex; i++) {
            it.next();
            it.remove();
        }
    }
   ```

   toIndex - fromIndex 횟수만큼 next(), remove()가 반복되므로 $O(N^2)$

   List를 상속받아 클래스를 설계하는 개발자는 다음처럼 removeRange를 오버라이드해서 clear의 성능을 높일 수 있음

   ```java
   @Override
    protected void removeRange(int fromIndex, int toIndex) {
        System.arraycopy(data, toIndex, data, fromIndex, size - toIndex);
        size -= (toIndex - fromIndex);
    }
   ```

3. 상속용 클래스를 시험하는 방법은 직접 하위 클래스를 만들어보는 것이 유일하다.

   꼭 필요한 protected 멤버를 놓쳤다면 하위 클래스를 작성할 때 알아챌 것이고, 하위 클래스 여러 개를 만들 동안 protected 멤버를 사용하지 않았다면, private 이었어야 했을 것이다.

4. 상속용 클래스의 생성자는 직접적으로든 간접적으로든 재정의 가능 메서드를 호출해서는 안된다.

   상위 클래스의 생성자가 하위 클래스의 생성자보다 먼저 실행되므로, 하위 클래스에서 재정의한 메서드가 하위 클래스의 생성자보다 먼저 호출되게 된다.

   ex.

   ```java
   public class Super {
       // Broken - constructor invokes an overridable method
       public Super() {
           overrideMe();
       }

       public void overrideMe() {
       }
   }
   ```

   ```java
   public final class Sub extends Super {
      // Blank final, set by constructor
      private final Instant instant;

       Sub() {
          instant = Instant.now();
       }

       // Overriding method invoked by superclass constructor
       @Override public void overrideMe() {
           System.out.println(instant);
       }

       public static void main(String[] args) {
           Sub sub = new Sub();
           sub.overrideMe();
       }
   }
   ```

   프로그램이 instant 두 번 출력할 것이라 기대하지만, 첫 번째는 null을 출력한다.

   상위 클래스의 생성자는 하위 클래스의 생성자가 인스턴스 필드를 초기화 하기도 전에 `overrideMe` 를 호출하기 때문이다

   clone이나 readObject 메서드는 생성자와 비슷한 효과를 내므로

   clone과 readObject도 재정의 가능한 메서드를 호출해서는 안된다.

5. 상속용으로 설계하지 않은 클래스는 상속을 금지하는 것이 좋다.

   1. 클래스를 final로 선언
   2. 모든 생성자를 private이나 package-private으로 선언하고 public 정적 팩토리를 만들어준다

   상속을 허용해야겠다면 클래스 내부에서는 재정의 가능 메서드를 사용하지 않도록 만들고, 문서에 이 사실을 남겨라
