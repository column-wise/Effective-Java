# item 22. 인터페이스는 타입을 정의하는 용도로만 사용하라

클래스가 어떤 인터페이스를 구현한다는 것은 자신의 인스턴스로 무엇을 할 수 있는지를 클라이언트에 얘기해주는 것이다.

이 지침에 맞지 않는 예로 상수 인터페이스라는 것이 있는데,

```java
// Constant interface antipattern - do not use!
public interface PhysicalConstants {
    static final double PI = 3.141592;
    // Avogadro's number (1/mol)
    static final double AVOGADROS_NUMBER = 6.022_140_857e23;
    // Boltzmann constant (J/K)
    static final double BOLTZMANN_CONSTANT = 1.380_648_52e-23;
    // Mass of the electron (kg)
    static final double ELECTRON_MASS = 9.109_383_56e-31;
}
```

메서드 없이, 상수를 뜻하는 static final 필드로 가득찬 인터페이스를 말한다.

이것은 인터페이스를 잘못 사용한 예이다.

- 클래스가 이 인터페이스를 implements 하면 내부 구현인 상수가 API에 노출됨.

  ```java
  public class MyClass implements PhysicalConstants { }
  ```

  해버리면 MyClass.AVOGADROS_NUMBER가 마치 MyClass의 public 멤버인 것처럼 노출됨

- 이 상수가 필요 없게 되더라도 바이너리 호환성을 위해 인터페이스를 계속 구현해야 함.

  바이너리 호환성: 이미 컴파일된(바이트 코드로 만들어진) 다른 코드가 수정된 새 버전의 클래스를 재컴파일 없이 그대로 사용할 수 있는 성질

  만약 버전 1에서

  ```java
  public class MyClass implements PhysicalConstants { /* ... */ }
  ```

  였는데, 버전 2에서 PI를 쓰지 않게 되었다고 해서

  ```java
  // 인터페이스 implements 삭제
  public class MyClass { /* ... */ }
  ```

  라고 바꾸면,
  버전 1에 컴파일된 다른 코드(예: MyClass.PI)는
  런타임에 MyClass에 PI가 없다고 오류(UnsatisfiedLinkError 등)를 일으킨다..

- final이 아닌 클래스가 이 상수 인터페이스를 구현한다면, 모든 하위 클래스의 이름공간이 이 인터페이스가 정의한 상수들로 오염되어 버린다.

  ```java
  public interface PhysicalConstants { double PI = 3.14; }
  public class Base implements PhysicalConstants { }
  public class Derived extends Base { }
  ```

  Derived.PI 가 유효함.  
  하위 클래스들에서 모두 PI 등의 상수 이름을 자기 멤버처럼 갖게 됨

---

<br>
상수를 공개할 목적이라면

1. 그 클래스나 인터페이스 자체에 추가

   Integer나 Double에 선언된 MAX_VALUE, MIN_VALUE 등

2. Enum 으로 만든다

3. 인스턴스화 할 수 없는 유틸리티 클래스로 만든다.

   ```java
   // Constant utility class
   package com.effectivejava.science;
   public class PhysicalConstants {
       private PhysicalConstants() { } // Prevents instantiation
       public static final double AVOGADROS_NUMBER = 6.022_140_857e23;
       public static final double BOLTZMANN_CONST = 1.380_648_52e-23;
       public static final double ELECTRON_MASS = 9.109_383_56e-31;
   }
   ```

   PhysicalConstants.AVOGADROS_NUMBER 처럼 유틸리티 클래스의 상수를 너무 자주 사용한다면 정적 임포트로 클래스 이름은 생략할 수 있다.

   ```java
   // Use of static import to avoid qualifying constants
   import static com.effectivejava.science.PhysicalConstants.*;
   public class Test {
       double atoms(double mols) {
            return AVOGADROS_NUMBER * mols;
        }
       ...
       // Many more uses of PhysicalConstants justify static import
   }
   ```
