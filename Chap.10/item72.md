# item 72. 표준 예외를 사용하라

## 표준 예외를 재사용하면 얻는게 많다.

1. 여러분의 API가 다른 사람이 익히고 사용하기 쉬워진다.
2. 예외 클래스 수가 적을수록 메모리 사용량도 줄고, 클래스를 적재하는 시간도 적게 걸린다.

## 재사용하기 좋은 표준 예외

1. `IllegalArgumentException`

   호출자가 인수로 부적절한 값을 넘길 때 던지는 예외

   ※ null 값을 허용하지 않는 메서드에 null을 건네거나 시퀀스의 허용 범위를 넘는 값을 건넬 때 등

   더 구체적으로 표현할 수 있는 예외가 있는 상황(`NullPointerException`, `IndexOutOfBoundsException`)에서는 `IllegalArgumentException`으로 뭉뚱그리지 말고 구분해서 사용하자

2. `IllegalStateException`

   대상 객체의 상태가 호출된 메서드를 수행하기에 적합하지 않을 때

   ex. 제대로 초기화되지 않은 객체를 사용하려 할 때

3. `ConcurrentModificationException`

   단일 쓰레드에서 사용하려고 설계한 객체를 여러 쓰레드가 동시에 수정하려 할 때 사용

4. `UnsupportedOperationException`

   클라이언트가 요청한 동작을 대상 객체가 지원하지 않을 때 사용

5. 다른 예외

   위 예외만 사용하라는 뜻은 아니다.

   예를 들어 복소수나 유리수를 다루는 객체를 작성한다면  
   `ArithmeticException`이나 `NumberFormatException` 등도 사용할 수 있을 것이다.

## 정리

1. 표준 예외를 적극적으로 재사용하면 코드 가독성과 유지보수성이 올라간다.

2. 단, 재사용은 의미 기반으로, 직접 구현은 꼭 필요할 때만 하자.

3. 예외 선택이 애매할 때는 "호출자의 잘못 vs 객체의 상태" 로 구분해서 판단하자.
