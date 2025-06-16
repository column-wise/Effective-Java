# item 15. 클래스와 멤버의 접근 권한을 최소화하라

## 정보 은닉, 캡슐화
잘 설계된 컴포넌트는 모든 내부 구현을 숨겨, 구현과 API를 깔끔히 분리한다.  
API를 통해서만 다른 컴포넌트와 소통하며, 서로의 내부 동작은 신경쓰지 않는다.

### 정보 은닉의 장점
* 시스템 개발 속도를 높인다. 여러 컴포넌트를 병렬로 개발할 수 있기 때문
* 시스템 관리 비용을 낮춘다. 각 컴포넌트를 더 빨리 파악하여 디버깅할 수 있고, 다른 컴포넌트로 교체하는 부담도 적다.
* 성능 최적화가 쉽다. 다른 컴포넌트에 영향을 주지 않고, 해당 컴포넌트만 최적화할 수 있기 때문이다.
* 소프트웨어 재사용성을 높인다. 외부 의존 없이 독자적으로 동작할 수 있는 컴포넌트라면 낯선 환경에서도 유용하게 쓰일 가능성이 높기 때문
* 큰 시스템을 제작하는 난이도를 낮춰준다. 시스템 전체가 완성되지 않은 상태에서도 개별 컴포넌트의 동작을 테스트할 수 있기 때문이다.

### 접근 제한자
접근 제한자를 제대로 활용하는 것이 정보 은닉의 핵심이다.

**모든 클래스와 멤버의 접근성을 가능한 한 좁혀야 한다.**  
올바로 동작하는 한 항상 가장 낮은 접근 수준을 부여해야 한다.

#### 접근 제한자의 종류
* private: 멤버를 선언한 톱레벨 클래스에서만 접근할 수 있다.
* package-private(default): 멤버가 소속된 패키지 안의 모든 클래스에서 접근할 수 있다. 접근 제한자를 명시하지 않았을 때 기본 적용.(단, 인터페이스의 멤버는 기본적으로 public이 적용된다.)
* protected: 이 멤버를 선언한 클래스의 하위 클래스에서도 접근할 수 있다.
* public: 모든 곳에서 접근할 수 있다.
---

톱레벨 클래스와 인터페이스에 부여할 수 있는 접근 수준은 default나 public 두 가지다.

톱레벨 클래스나 인터페이스를 public으로 선언하면 공개 API가 되며, default로 선언하면 해당 패키지 안에서만 이용할 수 있다. 

패키지 외부에서 쓸 이유가 없다면 package-private(default)으로 선언하자. 그러면 언제든 수정, 교체, 제거할 수 있다.

#### 접근 제한자 예시
```java
// 📁 com.example.library
package com.example.library;

public class PublicTopLevelClass {
    public int publicField;
    protected int protectedField;
    int packagePrivateField; // default
    private int privateField;

    public void accessDemo() {
        // 같은 클래스 내부이므로 모든 필드 접근 가능
        System.out.println(publicField);
        System.out.println(protectedField);
        System.out.println(packagePrivateField);
        System.out.println(privateField);
    }
}

class PackagePrivateTopLevelClass {
    // 이 클래스는 com.example.library 패키지 밖에서는 접근 불가
}
```
```java
// 📁 com.example.library.sub
package com.example.library.sub;

import com.example.library.PublicTopLevelClass;

public class SubClass extends PublicTopLevelClass {
    public void testAccess() {
        System.out.println(publicField);         // ✅ 가능 (public)
        System.out.println(protectedField);      // ✅ 가능 (protected: 상속관계 + 다른 패키지)
        // System.out.println(packagePrivateField); // ❌ 불가 (다른 패키지)
        // System.out.println(privateField);         // ❌ 불가 (private)
    }
}
```

SubClass는 필드들이 정의된 PublicTopLevelClass와 다른 패키지이지만, 상속 받았기 때문에 protectedField에 접근이 가능

```java
// 📁 com.example.other
package com.example.other;

import com.example.library.PublicTopLevelClass;

public class OtherClass {
    public void testAccess() {
        PublicTopLevelClass obj = new PublicTopLevelClass();
        System.out.println(obj.publicField);         // ✅ 가능
        // System.out.println(obj.protectedField);   // ❌ 불가 (상속 아님)
        // System.out.println(obj.packagePrivateField); // ❌ 불가 (다른 패키지)
        // System.out.println(obj.privateField);     // ❌ 불가 (private)
    }
}
```

OtherClass는 PublicTopLevelClass 상속도 받지 않았고, 다른 패키지에 위치하기 때문에 publicField에만 접근 가능

---

### 꿀팁
1. 클래스의 공개 API를 설계한 후, 그 외의 모든 멤버는 private으로 선언. 같은 패키지의 다른 클래스가 접근해야 하는 멤버에 한하여 package-private(default)로 풀어준다.

2. 권한을 풀어주는 일을 너무 자주하게 된다면 컴포넌트를 더 분해해야 하는 것은 아닌지 고민하자.

3. 상위 클래스의 메서드를 재정의할 때는 상위 클래스의 메서드보다 접근 수준을 좁게 설정할 수 없다.(Liskov Substitution Principle: 상위 클래스의 인스턴스는 하위 클래스의 인스턴스로 대체해 사용할 수 있어야 한다.) 

    인터페이스의 메서드들은 암묵적으로 public이기 때문에 이 인터페이스를 구현하는 클래스도 메서드를 public으로 선언해야 함

4. public 클래스의 인스턴스 필드는 되도록 public이 아니어야 한다. 

    필드를 public으로 선언하면 Setter 없이 누구나 값을 바꿀 수 있게 되므로 클래스의 일관성이 무너짐. 그리고 필드 변경 시 값을 로깅하거나 동기화하거나, 락을 획득하는 작업등을 수행할 수 없으므로 public 가변 필드를 갖는 클래스는 일반적으로 Thread-Safe하지 않다.
    ```java
    public class Point {
    private int x;
    private int y;

        public int getX() { return x; }
        public void setX(int x) { 
            // 예외 검사나 로깅 가능
            this.x = x; 
        }
    }
    ```

    구현 세부사항에 대한 유연성도 상실한다.
    필드를 public으로 두면 외부 API로 고정되고, 이후 내부 구현을 변경하고 싶더라도 외부 코드에서 참조하고 있을 수 있으므로 변경이 힘들다.
    ```java
    public final List<String> names = new ArrayList<>();
    // 이 필드를 없애고 Map으로 바꾸고 싶어도 못 바꿈
    ```

5. 클래스 내의 상수가 해당 클래스가 표현하는 추상 개념을 완성하는데 꼭 필요하다면 public static final 필드로 공개해도 된다.
    ```java
    public final class MathConstants {
    private MathConstants() {}
    public static final double PI = 3.14159;
    public static final double E = 2.71828;
    }

    public final class AppConfig {
        private AppConfig() {}
        public static final String APP_NAME = "YOITTANG";
        public static final int MAX_USER = 1000;
    }
    ```

    관례상 이런 상수의 이름은 대문자 알파벳으로 쓰며, 단어 사이에 밑줄을 넣는다. (APP_NAME 등)  

    이런 필드는 반드시 기본 타입 값이나 불변 객체를 참조해야 한다.  
    가변 객체(예: ArrayList, Date, 사용자 정의 클래스 등) 참조 자체는 바뀌지 않지만 객체 내용은 클래스 밖에서 수정할 수 있기 때문이다.

    ```java
    // 위험! 외부에서 배열 요소를 바꿀 수 있음
    public static final Thing[] VALUES = { new Thing("A"), new Thing("B") };
    ```

    ```java
    // 공격/버그 예시
    ThingConstants.VALUES[0] = new Thing("HACKED");
    // 라이브러리 내부 불변식, 보안 정책 모두 붕괴
    ```

    **-> 해결 방법**

    1. 불변 리스트로 감싸기
        ```java
        private static final Thing[] PRIVATE_VALUES = { ... };

        public static final List<Thing> VALUES =
            Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES));
        ```
        
        기존 코드에서 public static final Thing[] VALUES를
        private static으로 수정, PRIVATE_VALUES를 수정 불가능하도록 Collections.unmodifiableList로 변환

        but Things 객체가 가변이면 VALUES.get(0).setName("Hack") 같은 내부 조작은 여전히 가능함

    2. 복사본(clone) 반환 메서드 사용
        ```java
        private static final Thing[] PRIVATE_VALUES = { ... };

        public static Thing[] values() {
            return PRIVATE_VALUES.clone();
        }
        ```

