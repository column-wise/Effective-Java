# item 53. 가변인수는 신중히 사용하라

가변인수(varargs) 메서드는 명시한 타입의 인수를 0개 이상 받을 수 있다.  
가변인수 메서들 호출하면, 가장 먼저 인수의 개수와 길이가 같은 배열을 만들고, 인수들을 배열에 저장하여 가변인수 메서드에 건네준다.

ex. 간단한 가변인수 메서드

```java
static int sum(int... args) {
    int sum = 0;
    for (int arg : args)
        sum += arg;
    return sum;
}
```

ex. 인수가 1개 이상이어야 하는 가변인수 메서드(잘못된 예시)

```java
// The WRONG way to use varargs to pass one or more arguments!
static int min(int... args) {
    if (args.length == 0)
        throw new IllegalArgumentException("Too few arguments");
    int min = args[0];
    for (int i = 1; i < args.length; i++)
        if (args[i] < min)
            min = args[i];
    return min;
}
```

인수를 0개만 넣어 호출하면 런타임(!)에 실패하게 된다.

ex. 인수가 1개 이상이어야 할 때 가변인수를 제대로 사용하는 방법

```java
// The right way to use varargs to pass one or more arguments
static int min(int firstArg, int... remainingArgs) {
    int min = firstArg;
    for (int arg : remainingArgs)
        if (arg < min)
            min = arg;
    return min;
}
```

첫 번째로 평범한 매개변수를 받고, 가변인수는 두 번째로 받으면 문제가 없다.

---

가변인수 메서드는 호출될 때마다 배열을 새로 하나 할당하고 초기화하기 때문에  
성능에 민감한 상황이라면 가변인수가 걸림돌이 될 수 있다.

이럴 때 사용할 수 있는 패턴이 있는데,

```java
public void foo() { }
public void foo(int a1) { }
public void foo(int a1, int a2) { }
public void foo(int a1, int a2, int a3) { }
public void foo(int a1, int a2, int a3, int... rest) { }
```

만약 foo라는 메서드 호출의 95%가 인수를 3개 이하로 갖는다고 해보자.
그럼 단 5%의 메서드 호출에만 배열이 생성되므로 성능 최적화를 할 수 있다.
