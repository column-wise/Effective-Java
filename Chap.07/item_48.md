# item 48. 스트림 병렬화는 주의해서 적용하라

자바는 처음 릴리스부터 스레드, 동기화, wait/notify를 지원했고,  
자바 5부터는 동시성 컬렉션인 java.util.concurrent 라이브러리와 실행자(Executor) 프레임워크를 지원했다.  
자바 7부터는 포크-조인 패키지를 추가했다.

자바 8부터는 parallel 메서드만 한 번 호출하면 파이프라인을 병렬 실행할 수 있는 스트림을 지원했다.

자바로 동시성 프로그램을 작성하는 것은 쉬운 일이지만, 이를 올바르고 빠르게 작성하기는 여전히 어렵다.

## 병렬 스트림 파이프라인

```java
import java.math.BigInteger;
import java.util.stream.Stream;

// Stream-based program to generate the first 20 Mersenne primes
public class MersennePrimes {
    private static final BigInteger ONE = BigInteger.ONE;
    private static final BigInteger TWO = BigInteger.TWO;

    public static void main(String[] args) {
        primes()
            .map(p -> TWO.pow(p.intValueExact()).subtract(ONE))
            .filter(mersenne -> mersenne.isProbablePrime(50))
            .limit(20)
            .forEach(System.out::println);
    }

    static Stream<BigInteger> primes() {
        return Stream.iterate(TWO, BigInteger::nextProbablePrime);
    }
}
```

아이템 45에서 다뤘던 메르센 소수를 생성하는 프로그램이다.  
이 프로그램의 속도를 높이고 싶어 스트림 파이프라인의 parallel()을 호출하면 liveness failure(프로세스가 무한정 기다리는 현상)가 발생하는데,

이것은 스트림 라이브러리가 이 파이프라인을 병렬화하는 방법을 찾지 못했기 때문이다.

데이터 소스가 Stream.iterate거나 중간 연산으로 limit을 사용하면 파이프라인 병렬화로는 성능 개선을 기대할 수 없다.

거기에 파이프라인 병렬화는 limit을 다룰 때 CPU 코어가 여유가 있으면 원소를 추가로 몇 개 더 처리한 후 제한된 개수 이후의 결과를 버리는데,  
다음 메르센 소수를 찾는 비용이 이전까지의 원소 전부를 계산한 비용을 합친 만큼 들어 parallel이 제 기능을 못하게 된다.

## 스트림 병렬화에 영향을 미치는 요소

### 데이터 스트림의 소스

스트림 파이프라인 병렬화는 스트림의 소스가 ArrayList, HashMap, HashSet, ConcurrentHashmap의 인스턴스거나 배열, int 범위, long 범위일 때 효과가 좋다.

이 자료구조들은 Spliterator가 데이터를 원하는 크기로 정확하고 쉽게 나눠 다수의 쓰레드에 할당하기가 좋다는 특징이 있다.

또한 이 자료구조들은 원소들을 순차적으로 실행할 때 참조 지역성이 뛰어나다.

### 종단 연산의 동작 방식

종단 연산이 수행하는 작업량이 파이프라인 전체 작업 중 상당 부분을 차지하면서 순차적인 연산이라면  
파이프라인 병렬 수행의 효율은 떨어진다.

종단 연산 중 병렬화에 가장 적절한 것은 축소(reduction)이다.

축소는 파이프라인에서 만들어진 모든 원소를 하나로 합치는 작업으로, Stream의 reduce 메서드 중 하나, min, max, count, sum 등이다.

anyMatch, allMatch, noneMatch 등 조건에 맞으면 바로 반환되는 메서드도 병렬화에 적합하다.

반면, 가변 축소(mutable reduction)을 수행하는 Stream의 collect 메서드는 병렬화에 적합하지 않다.  
컬렉션을 합치는 부담이 크기 때문이다.

### spliterator

직접 구현한 Stream, Iterable, Collection이 효율 높은 병렬화을 지원하도록 하고 싶다면 spliterator 메서드를 반드시 재정의하고, 결과 스트림의 병렬화 성능을 강도 높게 테스트 해야 한다.

## 병렬화를 잘못 쓰면

성능이 나빠질 뿐만 아니라 결과 자체가 잘못되거나 예상 못한 동작이 발생할 수 있다.

심지어 데이터 소스 스트림이 효율적으로 나눠지고, 빨리 끝나는 종단 연산을 사용하고, 함수 객체들도 간섭하지 않더라도  
파이프라인이 수행하는 작업이 병렬화에 드는 추가 비용을 상쇄하지 못한다면 성능 향상은 미미할 수 있다.

스트림 안의 원소 수와 원소당 수행되는 코드 줄 수를 곱해 최소 수십만은 되어야 성능 향상을 기대할 수 있다.(Lea14)

병렬화 도입 전, 후로 반드시 성능을 테스트해야 한다.

## 병렬화를 잘 쓰면

조건이 잘 갖춰지면 parallel 메서드 호출 하나로 거의 프로세서 코어 수에 비례하는 성능 향상을 볼 수 있다.

```java
// Prime-counting stream pipeline - benefits from parallelization
static long pi(long n) {
    return LongStream.rangeClosed(2, n)
        .mapToObj(BigInteger::valueOf)
        .filter(i -> i.isProbablePrime(50))
        .count();
}
```

n 이하 소수의 개수를 계산하는 함수이다.

```java
// Prime-counting stream pipeline - parallel version
static long pi(long n) {
    return LongStream.rangeClosed(2, n)
        .parallel()
        .mapToObj(BigInteger::valueOf)
        .filter(i -> i.isProbablePrime(50))
        .count();
}
```

작가의 pc에서는 병렬화 덕에 3.37배가 빨라졌다고 한다.