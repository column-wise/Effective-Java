# item 23. 태그 달린 클래스보다는 클래스 계층 구조를 활용하라

## Tagged Class

```java
// Tagged class - vastly inferior to a class hierarchy!
class Figure {
    enum Shape { RECTANGLE, CIRCLE };
    // Tag field - the shape of this figure
    final Shape shape;
    // These fields are used only if shape is RECTANGLE
    double length;
    double width;
    // This field is used only if shape is CIRCLE
    double radius;

    // Constructor for circle
    Figure(double radius) {
        shape = Shape.CIRCLE;
        this.radius = radius;
    }

    // Constructor for rectangle
    Figure(double length, double width) {
        shape = Shape.RECTANGLE;
        this.length = length;
        this.width = width;
    }

    double area() {
        switch (shape) {
            case RECTANGLE:
                return length * width;
            case CIRCLE:
                return Math.PI * (radius * radius);
            default:
                throw new AssertionError(shape);
        }
    }
}
```

하나의 클래스 안에서 여러 도형 타입을 표현

Shape라는 enum으로 도형 종류 정의  
각 Figure 인스턴스는 shape 필드로 자신이 어떤 도형인지 표시함

length, width는 사각형일때만 쓰이고, radius는 원일때만 쓰임.  
-> 불필요한 필드들이 공존해서 메모리 낭비

필드들을 final로 선언하려면 해당 모양에서 쓰이지 않는 필드들까지 생성자에서 초기화 해야 함.

area() 메서드에서는 switch로 모양에 따라 분기하고 있는데, 새 모양을 추가할 때마다 switch 문을 고쳐야 함

=> ⚠️ **tagged 클래스는 클래스 계층 구조를 어설프게 흉내낸 아류, 장황하고 오류를 내기 쉽고, 비효율적이다 !!!!!!!!** ⚠️

## Class Hierarchy

1. 계층구조의 root가 될 추상 클래스를 정의하고, 태그 값에 따라 동작이 달라지는 메서드들을 루트 클래스의 추상 메서드로 선언한다.
   area()가 그 대상이다

2. 태그 값에 상관 없이 동작이 일정한 메서드는 루트 클래스의 일반 메서드로 추가,  
   모든 하위 클래스에서 공통으로 사용하는 데이터 필드들도 전부 루트로 올린다.

3. 루트 클래스를 확장한 구체 클래스를 하나씩 정의한다.  
   Figure를 확장한 Circle, Rectangle

```java
// Class hierarchy replacement for a tagged class
abstract class Figure {
    abstract double area();
}

class Circle extends Figure {
    final double radius;
    Circle(double radius) { this.radius = radius; }
    @Override
    double area() {
        return Math.PI * (radius * radius);
    }
}

class Rectangle extends Figure {
    final double length;
    final double width;
    Rectangle(double length, double width) {
        this.length = length;
        this.width = width;
    }
    @Override
    double area() {
        return length * width;
    }
}
```
