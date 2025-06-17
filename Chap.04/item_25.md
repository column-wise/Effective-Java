# item 25. 톱레벨 클래스는 한 파일에 하나만 담으라

소스파일 하나에 톱레벨 클래스를 여러 개 선언하더라도 자바 컴파일러는 오류를 내지 않는다.

## 잘못된 예시

```java
public class Main {
    public static void main(String[] args) {
        System.out.println(Utensil.NAME + Dessert.NAME); // "pancake"
    }
}
```

두 클래스를 `Utensil.java` 라는 한 파일 안에 정의했다고 해보자.

```java
class Utensil {
    static final String NAME = "pan";
}

class Dessert {
    static final String NAME = "cake";
}
```

출력:

```
pancake
```

그럼 이제 같은 클래스를 담은 `Dessert.java` 파일을 만들었다고 해보자..

```java
class Utensil {
    static final String NAME = "pot";
}

class Dessert {
    static final String NAME = "pie";
}
```

`javac Main.java Dessert.java` 명령으로 컴파일 하면  
컴파일러는 `Main.java`를 읽다가 Utensil을 참조하므로 `Utensil.java`를 읽음.

그 안에 `Dessert` 정의가 들어있음.

그런데 커맨드 라인에 `Dessert.java` 도 따로 명시했기 때문에 클래스 중복 정의 오류 발생

## 해결법

톱레벨 클래스들(Utensil과 Dessert)을 서로 다른 소스 파일로 분리하면 끝

굳이 한 파일에 담고 싶다면 정적 멤버 클래스를 사용(static inner class는 outer 클래스와 독립적으로 존재할 수 있으니까)

```java
public class Test {
    public static void main(String[] args) {
        System.out.println(Utensil.NAME + Dessert.NAME); // 출력: pancake
    }

    private static class Utensil {
        static final String NAME = "pan";
    }

    private static class Dessert {
        static final String NAME = "cake";
    }
}
```
