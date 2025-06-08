# item 06. 불필요한 객체 생성을 피하라

객체 하나를 재사용하는 편이 나을 때가 많다.

```java
String s = new String("bikini"); // 절대 따라하지 말 것 !
```

^ 위 문장은 실행될 때마다 String 인스턴스를 새로 만든다.

```java
String s = "bikini";
```

^ 이 코드는 새로운 인스턴스를 매 번 만드는 대신 하나의 String 인스턴스를 사용한다.  
같은 가상 머신 안에서 이와 똑같은 문자열 리터럴을사용하는 모든 코드가 같은 객체를 재사용함이 보장된다.

<br>

생성 비용이 아주 비싼 객체도 있을 수 있다.  
예를 들어 주어진 문자열이 유효한 로마 숫자인지를 확인하는 메서드를 작성해보면

```java
static boolean isRomanNumeral(String s) {
    return s.matches(
        "^(?=.)M*(C[MD]|D?C{0,3})"
        + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$"
        )
}
```

`String.matches`는 정규표현식으로 문자열 형태를 확인하는 가장 쉬운 방법이지만,
이 메서드가 내부에서 만드는 정규표현식용 `Pattern` 인스턴스는 한 번 쓰고 버려져 GC의 대상이 된다.  
`Pattern`은 입력받은 정규표현식에 해당하는 유한 상태 머신(finite state machine)을 만들기 때문에 인스턴스 생성 비용이 높다.

```java
public class RomanNumerals {
    private static final Pattern ROMAN = Pattern.compile(
        "^(?=.)M*(C[MD]|D?C{0,3})"
        + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$"
    );

    static boolean isRomanNumeral(String s) {
        return ROMAN.matcher(s).matches();
    }
}
```

개선된 `isRomanNumeral` 방식의 클래스가 초기화된 후 이 메서드를 한 번도 호출하지 않는다면 `ROMAN` 필드는 쓸데 없이 초기화 된 셈이지만, [지연 초기화](/Chap.09/item_67.md)를 권하지는 않는다.  
지연 초기화는 코드를 복잡하게 만드는데, 성능은 크게 개선되지 않을 때가 많기 때문이다.

<br>

객체가 불변이라면 재사용해도 안전함이 명백하다.  
[어댑터(Adapter / Wrapper)](https://refactoring.guru/ko/design-patterns/adapter) 패턴을 생각해보자.  
어댑터는 실제 작업은 뒷단 객체에 위임하고 자신은 제2의 인터페이스 역할을 해주는 객체다.  
뒷단 객체 외에는 관리할 상태가 없으므로 뒷단 객체 하나당 어댑터 하나씩만 만들어지면 충분하다.

`Map` 인터페이스의 `keySet` 메서드는 `Map` 객체 안의 키 전부를 담은 `Set`뷰를 반환한다.
`keySet`을 호출할 때마다 새로운 `Set` 인스턴스가 생길 필요가 없다.  
이 뷰는 `Map`에 연결된 객체이므로 반환한 객체 중 하나를 수정하면 다른 모든 객체가 따라서 바뀌어야 한다.

```java
import java.util.*;

public class KeySetViewExample {
    public static void main(String[] args) {
        Map<String, Integer> map = new HashMap<>();
        map.put("apple", 1);
        map.put("banana", 2);
        map.put("cherry", 3);

        Set<String> keys1 = map.keySet(); // 첫 번째 keySet view
        Set<String> keys2 = map.keySet(); // 두 번째 keySet view

        System.out.println("keys1 == keys2: " + (keys1 == keys2)); // true: 같은 인스턴스일 수 있음

        // keys1에서 제거하면 map에도 반영됨
        keys1.remove("banana");

        System.out.println("map: " + map);      // {apple=1, cherry=3}
        System.out.println("keys2: " + keys2);  // [apple, cherry] - keys2도 영향 받음
    }
}
```

`keySet()`은 view 객체이므로 메모리 절약을 위해 같은 인스턴스를 반환해도 문제 없음.

<br>

불필요한 객체를 만들어내는 또 다른 예로 오토 박싱(auto boxing)을 들 수 있다.  
[오토박싱은 기본 타입과 그에 대응하는 박싱된 기본 타입의 구분을 흐려주지만, 완전히 없애주는 것은 아니다.](/Chap.09/item_61.md)

```java
private static long sum() {
    Long sum = 0L;
    for (long i = 0; i <= Integer.MAX_VALUE; i++) {
        sum += i;
    }

    return sum;
}
```

`sum` 변수를 long이 아닌 Long으로 선언해서 불필요한 Long 인스턴스가 $2^{31}$개 생성되었다. (long 타입 `i`가 Long 타입인 `sum`에 더해질 때마다)

<br>

### 하지만 !

"객체 생성은 비싸니 피해야 한다" 로 이해하면 안된다..  
불필요하게 객체 생성을 할 필요는 없다. 재사용이 가능한 객체는 재사용하라 가 이 item의 핵심 내용

\+ DB connection 같은 무거운 객체가 아니라면 객체 풀 생성도 피하자  
객체 풀은 코드를 헷갈리게 만들고 메모리 사용량을 늘리고 성능을 떨어뜨린다.

## \+ Adapter Pattern

### 의도

클래스의 인터페이스를 사용자가 기대하는 인터페이스 형태로 적응(변환)시킴.  
서로 일치하지 않는 인터페이스를 같는 클래스들을 함께 동작하도록 함.

### 다른 이름

wrapper

### 예시

주식 시장 모니터링 앱을 만들고 있고, 이 앱은 여러 소스에서 주식 데이터를 XML 형식으로 다운로드한 후 사용자에게 보기 좋은 차트들과 다이어그램들을 표시

타사의 스마트 분석 라이브러리를 통합하여 당신의 앱을 개선하기로 결정  
but 이 분석 라이브러리는 JSON 형식의 데이터로만 작동한다.....

![](https://refactoring.guru/images/patterns/diagrams/adapter/problem-ko.png?id=33ecccae252bded5eb70e070ddf28633)

### 해결책

![](https://refactoring.guru/images/patterns/diagrams/adapter/solution-ko.png?id=73504c03a6e85f8b6182ad1701232d16)

어댑터를 만들어 해결할 수 있다.  
분석 라이브러리에 넘겨야 하는 XML 기반 객체들에 대해, JSON 형식을 흉내 내는 어댑터를 만들어 연동할 수 있다.

```java
class XmlToJsonAdapter {
    public JSONObject convert(String xml) {
        // xml 문자열을 JSONObject로 변환
    }
}
```

매 번 새로 만들 필요 없음.
