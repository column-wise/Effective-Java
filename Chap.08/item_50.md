# item 50. 적시에 방어적 복사본을 만들라

자바는 안전하다. 네이티브 메서드를 사용하지 않으니  
버퍼 오버런, 배열 오버런, 와일드 포인터 같은 메모리 충돌 오류에서 안전하다.

하지만! 그렇다고 하더라도  
클라이언트가 여러분의 불변식을 깨뜨리려 혈안이 되어 있다고 가정하고 방어적으로 프로그래밍 해야 한다.

```java
// Broken "immutable" time period class
public final class Period {
    private final Date start;
    private final Date end;

    /**
     * @param start the beginning of the period
     * @param end the end of the period; must not precede start
     * @throws IllegalArgumentException if start is after end
     * @throws NullPointerException if start or end is null
     */
    public Period(Date start, Date end) {
        if (start.compareTo(end) > 0)
            throw new IllegalArgumentException(start + " after " + end);
        this.start = start;
        this.end = end;
    }

    public Date start() {
        return start;
    }

    public Date end() {
        return end;
    }

    // ... Remainder omitted
}
```

클래스는 한 번 값이 정해지면 변하지 않도록 의도하였고, end는 start 보다 늦도록 한다는 제약을 두었다.

하지만!

```java
 // Attack the internals of a Period instance
Date start = new Date();
Date end = new Date();
Period p = new Period(start, end); 
end.setYear(78); // Modifies internals of p!
```

Date는 가변이기 때문에 불변식을 깨뜨릴 수 있다....

이를 막기 위해서는 Date 대신 불변인 Instant를 사용하면 된다.(LocalTimeZone이나 ZonedDateTime을 이용해도 됨.)

Date는 낡은 API이니 새로운 코드를 작성할 때는 더 이상 사용하면 안된다...

하지만 이미 많은 API와 내부 구현에 남아있으니 Date를 사용하는 코드도 안전하게 사용할 수 있도록 고쳐보자.

외부 공격으로부터 Period 인스턴스 내부를 보호하려면 생성자에서 받은 가변 매개변수 각각을 방어적으로 복사(defensive copy)해야 한다.

그리고 Period 인스턴스 안에서는 원본이 아닌 복사본을 사용한다.

```java
   // Repaired constructor - makes defensive copies of parameters
   public Period(Date start, Date end) {
       this.start = new Date(start.getTime());
       this.end   = new Date(end.getTime());
       if (this.start.compareTo(this.end) > 0)
         throw new IllegalArgumentException(
             this.start + " after " + this.end);
}
```

매개변수의 유효성을 검사하기 전에 방어적 복사본을 만들고, 이 복사본으로 유효성을 검사했다.  
이것은 멀티쓰레딩 환경에서는 원본 객체의 유효성을 검사하고 복사본을 만드는 찰나에 원본 객체를 수정할 수 있기 때문에 복사본을 먼저 만들고 유효성을 검사한 것이다.

방어적 복사를 할 때 Date에 clone 메서드를 사용하지 않은 것도 중요한데,  
Date 클래스는 final이 아니기 때문에 상속하여 악의적인 클래스를 만들 수 있기 때문이다.

이 하위 클래스에서 start와 end 필드의 참조를 담아두었다가 수정할 수 있도록 접근하는 길을 열어주게 될 수도 있다.

ex. EvilDate

```java
import java.util.*;

public class EvilDate extends Date {
    private static final List<Date> instances = new ArrayList<>();

    public EvilDate(long time) {
        super(time);
        instances.add(this); // 생성될 때마다 자기 자신을 리스트에 저장!
    }

    @Override
    public Object clone() {
        EvilDate copy = (EvilDate) super.clone();
        instances.add(copy); // 복제본도 저장
        return copy;
    }

    public static List<Date> getInstances() {
        return instances;
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        EvilDate evil = new EvilDate(System.currentTimeMillis());

        // Period는 Date.clone()으로 복사했다고 가정
        Period p = new Period(evil, evil); // 위험!

        // 나중에 외부에서 모든 Date 인스턴스를 가져올 수 있음
        for (Date d : EvilDate.getInstances()) {
            System.out.println("Leaked date: " + d);
        }
    }
}
```

period 내부에서 만약 방어적 복사를 수행하기 위해 clone()을 호출한다고 하면
```java
this.start = (Date) start.clone();  // 🚨 clone() 사용
this.end   = (Date) end.clone();
```

런타임에 stary, end는 EvilDate이므로 EvilDate의 clone이 호출되고, EvilDate의 static list에 copy된 Date 참조가 담긴다.

이것은 외부에서 접근 가능하므로 안전하지 않게 된다...

그래서 방어적 복사본을 만들 때 clone을 사용하면 안된다!

---

clone을 사용하지 않았더라도 아직 Period는 외부 공격에 취약한데,  
Period의 start(), end() 메서드가 필드를 드러내기 때문에

```java
 // Second attack on the internals of a Period instance
Date start = new Date();
Date end = new Date();
Period p = new Period(start, end); 
p.end().setYear(78); // Modifies internals of p!
```

여전히 공격 가능하다.

그래서 getter가 필드를 그대로 반환하지 않고, 가변 필드의 방어적 복사본을 만들어 반환하도록 하자.

```java

// Repaired accessors - make defensive copies of internal fields
   public Date start() {
       return new Date(start.getTime());
}
   public Date end() {
       return new Date(end.getTime());
}
```

이렇게 하면 Period는 완벽한 불변이 된다.

생성자와는 달리 접근자에서는 clone을 사용해도 되는데, 필드에 있는 Date 객체는 java.util.Date임이 확실하기 때문이다.

그렇더라도 item 13(clone 재정의는 주의해라)에서 설명한 이유 때문에 복사에는 생성자나 정적 팩토리를 쓰는게 좋다.

<br>

## 예외
모든 경우에서 방어적 복사를 수행해야 하는 것은 아니다.

	•	방어적 복사는 성능 비용이 있다. (clone(), new 등)
	•	모든 상황에서 복사를 정당화할 수 있는 건 아니다.
    •	클래스와 호출자가 같은 패키지 내에 있고, 내부 구현이 통제된 환경이라면, 방어적 복사를 생략해도 괜찮다.
    •	즉, “우린 서로 신뢰한다”는 암묵적 합의가 있는 경우
	•	“이제부터 이 객체는 내가 책임진다”는 의미를 가지는 메서드/생성자가 있음  
         → “소유권 이전(ownership transfer)” 패턴, 이런 경우는 문서에 정확히 밝혀야 함.  
         그리고 이런 클래스는 악의적인 클라이언트를 방어할 방법이 없으므로 피해가 클라이언트에만 국한되는 상황에서만 사용  
         ex. wrapper 클래스
    