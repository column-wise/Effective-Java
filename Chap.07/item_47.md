# item 47. 반환타입으로는 스트림보다 컬렉션이 낫다

자바 7까지는 일련의 원소를 반환하는 메서드의 리턴 타입으로 Collection, Iterable이나 배열을 사용했다.

자바 8에서 스트림이 추가되면서 메서드가 스트림을 반환할 수 있게 되었다.

하지만 스트림은 반복(iteration)을 지원하지 않으므로 사용자는 불편함을 겪을 것이다.  
for-each로 반복하고 싶다...

## 스트림에 담긴 원소를 반복해서 꺼낼 수 있도록 해보자

```java
// Won't compile, due to limitations on Java's type inference
for (ProcessHandle ph : ProcessHandle.allProcesses()::iterator) {
    // Process the process
}
```

하지만 이 코드는 컴파일되지 않는다.

```
Test.java:6: error: method reference not expected here
for (ProcessHandle ph : ProcessHandle.allProcesses()::iterator) {
                        ^

```

메서드 참조를 매개변수화된 Iterable로 형변환해줘야 한다.

```java
// Hideous workaround to iterate over a stream
for (ProcessHandle ph : (Iterable<ProcessHandle>) ProcessHandle.allProcesses()::iterator)
```

작동은 하지만 너무 난잡하고 직관성이 떨어진다.

어댑터 메서드를 사용하면 조금 나아지는데..

```java
// Adapter from Stream<E> to Iterable<E>
public static <E> Iterable<E> iterableOf(Stream<E> stream) {
    return stream::iterator;
}
```

```java
for (ProcessHandle p : iterableOf(ProcessHandle.allProcesses())) {
// Process the process
}
```

이렇게 하면 스트림을 for-each로 반복할 수 있다.

한편, API가 Iterable만 반환하면 이를 스트림 파이프라인에서 처리하려는 요구는 만족시킬 수 없을 것이다.

```java
// Adapter from Iterable<E> to Stream<E>
public static <E> Stream<E> streamOf(Iterable<E> iterable) {
    return StreamSupport.stream(iterable.spliterator(), false);
}
```

스트림 파이프라인을 사용하려는 사람과 반복문에서 쓰려는 사람 모두를 배려해 공개 API를 만들자.

Collection 인터페이스는 Iterable의 하위 타입이고, stream 메서드도 제공하므로 반복과 스트림으르 동시에 지원한다.  
원소 시퀀스를 반환하는 공개 API의 반환 타입에는 Collection이나 그 하위 타입을 쓰는게 일반적으로 최선이다.

하지만 반환하려는 원소 시퀀스가 너무 커서 메모리에 부담이 된다면 컬렉션보다는 스트림으로 지연 계산되도록 하는 것이 나을 수 있다.

반환할 시퀀스가 크지만 표현을 간결하게 할 수 있다면 전용 컬렉션을 구현하는 방안을 검토해보자.

ex. 주어진 집합의 멱집합을 반환하는 메서드

원소의 개수가 n 개이면, 멱집합의 원소 개수는 $2^n$개가 된다.

그러니 멱집합을 표준 컬렉션 구현체에 저장하려는 생각은 메모리에 부담이 되어 위험하다.  
하지만 AbstractList를 이용하면 전용 컬렉션을 구현하여 반환할 수 있다.

```java
// Returns the power set of an input set as custom collection
public class PowerSet {
    public static final <E> Collection<Set<E>> of(Set<E> s) {
        List<E> src = new ArrayList<>(s);
        if (src.size() > 30)
            throw new IllegalArgumentException("Set too big " + s);

        return new AbstractList<Set<E>>() {
            @Override
            public int size() {
                return 1 << src.size(); // 2 to the power srcSize
            }

            @Override
            public boolean contains(Object o) {
                return o instanceof Set && src.containsAll((Set<?>) o);
            }

            @Override
            public Set<E> get(int index) {
                Set<E> result = new HashSet<>();
                for (int i = 0; index != 0; i++, index >>= 1)
                    if ((index & 1) == 1)
                        result.add(src.get(i));
                return result;
            }
        };
    }
}
```

부분 집합의 각 원소를 비트 벡터로 표현된 인덱스로 취급
예: 집합 {a, b, c} (3개 원소)라면,

0 (000) → {}

1 (001) → {a}

2 (010) → {b}

...

7 (111) → {a, b, c}

총 2³ = 8개의 부분 집합이 만들어짐.

```java
import java.util.*;

public class PowerSetExample {
    public static void main(String[] args) {
        Set<String> original = Set.of("A", "B", "C");

        Collection<Set<String>> powerSet = PowerSet.of(original);

        System.out.println("Power set of " + original + ":");
        for (Set<String> subset : powerSet) {
            System.out.println(subset);
        }
    }
}
```

출력 결과  
->

```text
Power set of [A, B, C]:
[]
[A]
[B]
[A, B]
[C]
[A, C]
[B, C]
[A, B, C]
```

<br>

---

ex. 입력 리스트의 모든 부분리스트를 스트림으로 반환

(a, b, c)의 모든 부분 리스트는 다음과 같다:

prefixes:

(a)

(a, b)

(a, b, c)

각 prefix의 suffixes:

(a) → (a)

(a, b) → (a, b), (b)

(a, b, c) → (a, b, c), (b, c), (c)

결과:

(a), (a, b), (b), (a, b, c), (b, c), (c)

Collections.emptyList() 를 추가하면:

빈 리스트 []까지 포함해서 전체 부분 리스트 완성

```java
// Returns a stream of all the sublists of its input list
public class SubLists {
    public static <E> Stream<List<E>> of(List<E> list) {
        return Stream.concat(
            Stream.of(Collections.emptyList()),
            prefixes(list).flatMap(SubLists::suffixes)
        );
    }

    private static <E> Stream<List<E>> prefixes(List<E> list) {
        return IntStream.rangeClosed(1, list.size())
                        .mapToObj(end -> list.subList(0, end));
    }

    private static <E> Stream<List<E>> suffixes(List<E> list) {
        return IntStream.range(0, list.size())
                        .mapToObj(start -> list.subList(start, list.size()));
    }
}
```

```java
Stream.concat(Stream.of(Collections.emptyList()), ...);
```

로 빈 리스트를 맨 앞에 추가했다.

그리고 프리픽스들과 서픽스들의 스트림은 IntStream.range와 rangeClosed가 반환하는 연속된 정수 값들을 매핑해서 만들었다.

```java
for (int start = 0; start < src.size(); start++)
    for (int end = start + 1; end <= src.size(); end++)
        System.out.println(src.subList(start, end));

```

이런 이중 for문과 동일한 역할을 하므로  
이렇게 구현할 수도 있다.

```java
public static <E> Stream<List<E>> of(List<E> list) {
    return IntStream.range(0, list.size())
        .mapToObj(start ->
            IntStream.rangeClosed(start + 1, list.size())
                     .mapToObj(end -> list.subList(start, end)))
        .flatMap(x -> x);
}
```

더 간결하지만 가독성은 조금 떨어진다.......  
그리고 빈 리스트도 포함되지 않으므로 앞 코드터럼 concat을 붙이거나 rangeClosed 의 1을 (int)Math.signum(start)로 고쳐주면 된다.

## 정리

스트림을 반환하는 두 가지 구현을 알아봤는데, 반복을 사용하는 게 더 자연스러운 경우에는 어댑터를 써야 하므로 보기에 지저분하다.

가능한 상황이면 Collection을 반환.  
표준 컬렉션을 반환하는게 무난하고, 직접 만든 컬렉션도 고려할 만 하다.

그게 안되면 Stream이나 Iterable 중 자연스러운 것을 선택
