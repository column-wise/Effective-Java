# item 11. equals를 재정의하려거든 hashCode도 재정의하라

`equals`를 재정의한 클래스 모두에서 `hashCode`도 재정의해야 한다.

그렇지 않으면 hashCode 일반 규약을 어기게 되어 해당 클래스의 인스턴스를 `HashMap`이나 `HashSet` 같은 컬렉션의 원소로 사용할 때 문제를 일으킬 것이다.

## hashCode 규약

다음은 `Object` 명세에서 발췌한 규약이다.

> - `equals` 비교에 사용되는 정보가 변경되지 않았다면, 애플리케이션이 실행되는 동안 그 객체의 `hashCode` 메서드는 볓 번을 호출해도 일관되게 항상 같은 값을 반환해야 한다.  
>   단, 애플리케이션을 다시 실행한다면 이 값이 달라져도 상관 없다.
>
> - `equals(Object)`가 두 객체를 같다고 판단했다면, 두 객체의 `hashCode`는 똑같은 값을 반환해야 한다.
>
> - `equals(Object)`가 두 객체를 다르다고 판단했더라도, 두 객체의 `hashCode`가 서로 다른 값을 반환할 필요는 없다. 단, 다른 객체에 대해서는 다른 값을 반환해야 해시테이블의 성능이 좋아진다.

`hashCode` 재정의를 잘못했을 때 크게 문제가 되는 조항은 두 번째다.

논리적으로 같은 객체는 같은 해시코드를 반환해야 한다.

`equals`는 물리적으로 다른 두 객체를 논리적으로 같다고 할 수 있다.  
하지만 `Object`의 기본 `hashCode` 메서드는 이 둘이 전혀 다르다고 판단하여, 규약과 달리 서로 다른 값을 반환한다.

```java
Map<PhoneNumber, String> m = new HashMap<>();
m.put(new PhoneNumber(707, 867, 5309), "Jenny");
System.out.println(m.get(new PhoneNumber(707, 867, 5309), "Jenny"));  // null
```

여기에는 2개의 `PhoneNumber` 인스턴스가 사용됐다.  
하나는 `HashMap`에 "제니"를 넣을 때, 두 번째는 이를 꺼내려할 때 사용되었다.

두 인스턴스는 논리적으로 동치지만, 서로 다른 해시코드를 반환했기 때문에 get 메서드는 엉뚱한 해시 버킷에서 객체를 찾으려 한 것이다.

그렇다면 올바른 `hashCode` 메서드는 어떤 모습이어야 할까?

```java
// The worst possible legal hashCode implementation - never use!
@Override public int hashCode() { return 42; }
It’s legal because it ensures that equal objects have the same
```

적법하게 구현했지만(규약은 만족하지만) 절대 사용해서는 안되는 구현방법이다...

동치인 모든 객체에서 똑같은 해시코드를 반환하니 적법하다. 하지만 끔직하게도 모든 객체에게 똑같은 값만 내어주므로 모든 객체가 해시테이블의 버킷 하나에 담겨 마치 연결 리스트처럼 동작한다.

그 결과 평균 수행시간이 O(1)인 해시테이블이 O(n)으로 느려져서, 객체가 많아지면 도저히 쓸 수 없게 된다.

좋은 해시 함수라면 서로 다른 인스턴스에 다른 해시코드를 반환한다.

## 좋은 hashCode 작성법

1.  int 변수 result를 선언한 후 값 c로 초기화한다.

    이때 c는 해당 객체의 첫번째 핵심 필드를 단계 2.a 방식으로 계산한 해시코드다.  
    여기서 핵심 필드란 equals 비교에 사용되는 필드를 말한다.

2.  해당 객체의 나머지 핵심 필드 f 각각에 대해 다음 작업을 수행한다.

    a. 해당 필드의 해시코드 c를 계산한다.

        i. 기본 타입 필드라면, `Type.hashCode(f)`를 수행한다.
        여기서 `Type`은 해당 기본 타입의 박싱 클래스이다.

        ii. 참조 타입 필드면서 이 클래스의 `equals` 메서드가 이 필드의 `equals`를 재귀적으로 호출해 비교한다면,
        이 필드의 `hashCode`를 재귀적으로 호출한다.

        계산이 복잡해질 것 같다면, 이 필드의 표준형을 만들어 그 표준형의 `hashCode`를 호출한다.
        필드의 값이 null이라면 0을 사용한다.

        iii. 필드가 배열이라면, 핵심 원소 각각을 별도 필드처럼 다룬다.
        이상의 규칙을 재귀적으로 적용해 각 핵심 원소의 해시코드를 계산한 다음, 단계 2.b 방식으로 갱신한다.

        배열에 핵심 원소가 하나도 없다면 단순히 상수(0 추천)를 사용한다.

        모든 원소가 핵심 원소라면 `Arrays.hashCode`를 사용한다.

    b. 단계 2.a에서 계산 해시코드 c로 result를 갱신한다.

        result = 31 * result + c;

3.  result를 반환한다.

`hashCode`를 구현했다면 이 메서드가 동치인 인스턴스에 대해 똑같은 해시코드를 반환할지 확인하기 위해 단위 테스트를 작성하자...(AutoValue로 equals, hashCode 생성 시 생략)

파생 필드는 해시코드 계산에서 제외해도 된다.

또한 equals 비교에 사용되지 않은 필드는 **반드시** 제외해야 한다.  
그렇지 않으면 hashCode 규약 두 번째를 어기게 될 위험이 있다.

단계 2.b의 곱셈 31 \* result 는 필드를 곱하는 순서에 따라 result 값이 달라지게 한다.  
클래스에 비슷한 필드가 여러 개 일때 해시 효과를 크게 높여준다.

예를 들어 `String`의 `hashCode`를 곱셈 없이 구현한다면, 모든 아나그램의 해시코드가 같아진다.

31을 곱하는 이유는 홀수이면서 소수이기 때문이다.  
이 숫자가 짝수라면 오버플로 발생 시 정보를 더 많이 잃게 된다.  
2를 곱하는 것이 시프트 연산과 같은 결과를 내기 때문이다. (하위 비트가 무조건 0으로 채워짐)

`PhoneNumber` 클래스에 적용해보자.

```java
// Typical hashCode method
@Override public int hashCode() {
int result = Short.hashCode(areaCode);
    result = 31 * result + Short.hashCode(prefix);
    result = 31 * result + Short.hashCode(lineNum);
    return result;
}
```

단순하고, 충분히 빠르고, 서로 다른 전화번호들은 다른 해시 버킷들로 제법 훌륭히 분배해준다.

해시 충돌이 더욱 적은 방법을 찾고 싶다면 구아바의 `com.google.common.hash.Hashing`을 참고하라.

`Ojbect` 클래스는 임의의 개수만큼 객체를 받아 해시코드를 계산해주는 정적 메서드인 `hash`를 제공한다.  
이 메서드를 활용하면 앞의 요령대로 구현한 코드와 비슷한 수준의 `hashCode` 함수를 단 한 줄로 작성할 수 있다.  
하지만 아쉽게도 속도는 더 느리다.

```java
// One-line hashCode method - mediocre performance
@Override public int hashCode() {
    return Objects.hash(lineNum, prefix, areaCode);
}
```

## hashCode 값 캐싱

클래스가 불변이고, 해시코드를 계산하는 비용이 크다면, 매번 새로 계산하기보다는 캐싱하는 방식을 고려해야 한다.

이 타입의 객체가 주로 해시의 키로 사용될 것 같다면 인스턴스가 만들어질 때 해시코드를 계산해둬야 한다.

해시의 키로 사용되지 않는 경우라면 hashCode가 처음 불릴 때 계산하는 지연 초기화(Lazy Initialization) 전략을 사용해도 좋을 것이다.  
다만 그 클래스를 쓰레드 안전하게 만들도록 신경 써야 한다.

`PhoneNumber` 클래스는 굳이 이렇게까지 할 필요는 없지만 예시를 위해

```java
// hashCode method with lazily initialized cached hash code

private int hashCode; // Automatically initialized to 0

@Override public int hashCode() {
    int result = hashCode;

    if (result == 0) {
        result = Short.hashCode(areaCode);
        result = 31 * result + Short.hashCode(prefix);
        result = 31 * result + Short.hashCode(lineNum);
        hashCode = result;
    }

    return result;
}
```

`hashCode` 필드의 초깃값은 흔히 생성되는 객체의 해시코드와는 달라야 한다.

## 추가 조언

- 성능을 높이겠다고 해시코드 계산에 핵심 필드를 생략하면 안된다.

  속도는 빨라지겠지만, 해시 품질이 나빠져 해시테이블의 성능을 심각하게 떨어뜨릴 수 있다.

  실제로 Java 2 이전의 `String`은 최대 16개의 문자만으로 해시코드를 계산했는데, URL 처럼 계층적인 이름을 대량으로 사용한다면 앞서 이야기한 문제를 야기시킨다.

- hashCode가 반환하는 값의 생성 규칙을 API 사용자에게 자세히 공표하지 말아라.

  그래야 클라이언트가 이 값에 의지하지 않게 되고, 추후에 계산 방식을 바꿀 수 있다.
