# item 69. 예외는 진짜 예외 상황에만 사용하라

어쩌면 다음과 같은 코드를 마주칠지도 모른다...

```java
// Horrible abuse of exceptions. Don't ever do this!
try {
    int i = 0;
    while(true)
       range[i++].climb();
} catch (ArrayIndexOutOfBoundsException e) {
}
```

무슨 작업을 하는 코드인지 직관적으로 이해하기 어렵다...

배열의 원소를 순회하는 작업을 하는데, 다음과 같이 작성했다면 모든 자바 프로그래머가 바로 이해했을 것이다.

```java
for (Mountain m : range)
    m.climb();
```

예외를 써서 루프를 종료하도록 작성한 이유는 무엇이었을까....  
for 루프 안에는 i < range.length 같은 종료 조건이 있는데, 이 종료 조건 검사와 배열 인덱스에 접근할 때마다 배열의 범위 안에 있는지 체크하는 검사가 중복되므로  
종료 조건 검사 없이 while(true)로 돌다가 예외가 발생하면 던지도록 하는 것이 더 효율적이지 않나 라는 잘못된 생각 때문이었을 것이다.

실제로 예외를 사용한 쪽이 표준 관용구보다 훨씬 느렸다.

1. 예외는 명확한 검사(range[i], i < range.length 같은)보다 빠르게 만들어야 할 동기가 약하다.
2. 코드를 try-catch 블록 안에 넣으면 JVM이 적용할 수 있는 최적화가 제한된다.
3. 배열을 순회하는 표준 관용구는 앞서 걱정한 중복 검사를 수행하지 않는다. JVM이 알아서 최적화해 없애준다.

예외를 이용한 반복문은 성능을 떨어뜨린다 !!

또한 제대로 동작하지 않을 수도 있다.  
반복문 안에 버그가 숨어있다면 흐름 제어에 쓰인 예외가 이 버그를 숨겨 디버깅을 어렵게 만들 것이다.

=> 예외는 오직 예외 상황에서만 써야 한다. 절대로 일상적인 제어 흐름용으로 쓰여선 안 된다.

---

잘 설계된 API라면 클라이언트가 정상적인 제어 흐름에서 예외를 사용할 일이 없게 해야 한다.

ex. Iterator 인터페이스

```java
Iterator<Foo> i = collection.iterator();
while (i.hasNext()) {
    Foo foo = i.next();
    ...
}
```

Iterator에는 '상태 의존적' 메서드인 next를 호출해도 괜찮은지를 확인하는  
'상태 검사' 메서드 hasNext()가 있다.

hasNext가 없었다면

```java
try {
    Iterator<Foo> i = collection.iterator();
    while (true) {
        Foo foo = i.next();
        ...
    }
} catch (NoSuchElementException e) {
}
```

이런 방식은 장황하고, 헷갈리며, 속도도 느리고, 엉뚱한 곳에서 발생한 버그를 숨긴다.
