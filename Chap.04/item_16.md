# item 16. public 클래스에서는 public 필드가 아닌 접근자 메서드를 사용하라

인스턴스 필드를 모아놓는 일 외에는 아무일도 하지 않는 클래스를 작성할 때가 있다.
``` java
Class Point {
    public double x;
    public double y;
}
```
이런 클래스는 데이터 필드에 직접 접근할 수 있어 캡슐화의 이점을 제공하지 못한다.

객체지향 프로그래머는 필드들을 모두 private으로 바꾸고 public 접근자(Getter)를 추가한다.

```java
// Encapsulation of data by accessor methods and mutators
class Point {
    private double x;
    private double y;
    public Point(double x, double y) {
        this.x = x;
        this.y = y; 
    }
    public double getX() { return x; }
    public double getY() { return y; }
    public void setX(double x) { this.x = x; }
    public void setY(double y) { this.y = y; }
}
```

### 패키지 바깥에서 접근할 수 있는 클래스라면 접근자를 제공함으로써 클래스 내부 표현 방식을 언제든 바꿀 수 있는 유연성을 얻을 수 있다.

ex.
```java
public double getX() {
    return Math.round(x * 10.0) / 10.0; // 출력 형식 변경
}
```

⚠️ Getter로 감쌌다고 해서 시그니처를 변경해도 된다는 것은 아니다!!!!
double getX() -> float getX() 🚫🚫🚫

### 하지만 package-private 클래스 혹은 private 중첩 클래스라면 데이터 필드를 노출해도 문제 없다.

* package-private class
    ```java
    // 접근 제한자 생략 → 같은 패키지에서만 접근 가능
    class RGB {
        public int r, g, b;
    }
    ```
    * 이 클래스는 같은 패키지 안에서만 사용됨
    * 외부 API로 노출되지 않음
    * 필드를 굳이 숨길 필요 없다면, 단순화 차원에서 공개해도 좋다

* private nested class
    ```java
    public class Image {
        private static class Pixel {
            public int x, y;
            public int color;
        }
    }
    ```
    * Pixel은 Image 내부에서만 쓰임
    * 외부 클래스는 Pixel에 전혀 접근할 수 없음.
    * -> 필드를 공개해도 캡슐화가 깨지지 않음

### 그럼 public class의 필드가 final이면 직접 노출해도 문제가 없는가?
No No. 막 좋은 생각은 아님

여전히 API를 변경하지 않고는 표현 방식을 바꿀 수 없고, 필드를 읽을 때 부수 작업을 수행할 수 없다.

```java
public final class Time {
    public final int hour;
    public final int minute;

    public Time(int hour, int minute) {
        if (hour < 0 || hour >= 24)
            throw new IllegalArgumentException("Hour: " + hour);
        if (minute < 0 || minute >= 60)
            throw new IllegalArgumentException("Min: " + minute);

        this.hour = hour;
        this.minute = minute;
    }
}
```