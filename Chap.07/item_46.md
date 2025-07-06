# item 46. 스트림에서는 부작용 없는 함수를 사용하라

## 스트림 패러다임

- 외부 상태를 변경하지 말자(순수함수 사용)
- 중간 연산 과 종단 연산을 조합해 선언적으로 데이터 흐름을 기술
- 병렬 처리에 안전한 구조를 유지

※ 순수함수: 외부 상태에 의존하지도 않고, 변경시키지도 않는, 부수효과가 없는 함수

ex. 스트림의 잘못된 사용

```java
// Uses the streams API but not the paradigm--Don't do this!
Map<String, Long> freq = new HashMap<>();
try (Stream<String> words = new Scanner(file).tokens()) {
    words.forEach(word -> {
        freq.merge(word.toLowerCase(), 1L, Long::sum);
    });
}
```

외부 상태(Map)를 명시적으로 변경  
-> 병렬 처리에 안전하지 않고 스트림을 쓰는 이유(간결성, 병렬화, 가독성 등)을 잃게 됨  
-> 스트림 패러다임에 맞지 않음

```java
// Proper use of streams to initialize a frequency table
Map<String, Long> freq;
try (Stream<String> words = new Scanner(file).tokens()) {
    freq = words
        .collect(groupingBy(String::toLowerCase, counting()));
}
```

스트림 API를 제대로 사용하여 짧고 명확하다.

예시처럼 스트림의 forEach 연산은 병렬화 할 수도 없어 스트림 계산 결과를 보고할 때만 사용하고, 계산 시에는 사용하지 않는 것이 좋다.

## Collector

스트림을 사용하려면 꼭 알아야 하는 개념

java.util.stream.Collectors는 39개의 메서드를 가지고 있다.  
collector를 사용하면 스트림의 원소를 쉽게 컬렉션으로 모을 수 있다.

toList(), toSet(), toCollection(collectionFactory) 등등

### Collectors.toList()

ex. 빈도표에서 가장 흔한 단어 10개 추출

```java
// Pipeline to get a top-ten list of words from a frequency table
List<String> topTen = freq.keySet().stream()
    .sorted(comparing(freq::get).reversed())
    .limit(10)
    .collect(toList());
```

freq::get은 입력받은 단어를 빈도표에서 찾아 빈도를 반환, 흔한 단어가 위로 오게 reverse로 정렬

### Collectors.toMap()

한편 스트림을 맵으로 취합하는 기능을 보면,  
가장 간단한 맵 수집기는 toMap(kerMapper, valueMapper)

ex.

```java
// Using a toMap collector to make a map from string to enum
private static final Map<String, Operation> stringToEnum =
    Stream.of(values()).collect(
        toMap(Object::toString, e -> e));
```

values()는 Enum 클래스의 모든 상수들을 배열로 리턴하는 메서드

```java
enum Operation {
    ADD, SUBTRACT;

    private static final Map<String, Operation> stringToEnum =
        Stream.of(values()).collect(toMap(Object::toString, e -> e));

    public static Operation fromString(String symbol) {
        return stringToEnum.get(symbol);
    }
}
```

여기서 stringToEnum Map<String, Operation>은

```java
{
  "ADD" -> Operation.ADD,
  "SUBTRACT" -> Operation.SUBTRACT,
  ...
}
```

이런 꼴이 됨

Operation.fromString("ADD") -> Operation.ADD 가 반환됨

이런 간단한 toMap은 각 원소가 고유한 키를 사용할 때 유효하다.  
키가 겹치면 IllegalStateException을 던질 것

<br>

---

이런 충돌을 막기 위해 toMap에 병합함수(BinaryOperator\<U>)를 함께 넘길 수 있다.

ex1. 다양한 음악가의 앨법들을 담은 스트림을 가지고, 음악가와 그 음악가의 베스트 앨범을 연관 짓는 collector

```java
// Collector to generate a map from key to chosen element for key
Map<Artist, Album> topHits = albums.collect(
    toMap(Album::artist, a->a, maxBy(comparing(Album::sales))));
```

maxBy는 BinaryOperator에 정의된 정적 팩토리 메서드,  
Comparator\<T>를 인자로 받아 두 값을 비교하여 더 큰 값을 반환하는 BinaryOperator\<T>를 생성한다.

앨범 판매량을 비교하는 Comparator\<Album>을 만들고  
maxBy에서 받아 판매량이 더 큰 값을 반환하는 병합함수 생성

ex2. 마지막에 쓴 값을 취하는 수집기

```java
// Collector to impose last-write-wins policy
toMap(keyMapper, valueMapper, (v1, v2) -> v2)
```

<br>

---

네 번째 인수로 맵 팩토리를 받는 toMap도 있다.

toConcurrentMap은 병렬 실행된 후 결과로 ConcurrentHashMap 인스턴스를 생성하는 것도 있다

### groupingBy

이 메서드는 입력으로 분류 함수(classifier)를 받고 출력으로는 원소들을 카테고리별로 모아 놓은 맵을 담은 수집기를 반환한다.

이 카테고리가 해당 원소의 맵 키로 쓰인다.  
값은 해당 카테고리에 속하는 원소들

ex. item 45의 아나그램

```java
words.collect(groupingBy(word -> alphabetize(word)));
```

키는 알파벳화한 단어, 값은 알파벳화 결과가 같은 단어들의 리스트

<br>

---

groupingBy가 반환하는 수집기가 리스트 외의 값을 갖는 맵을 생성하도록 하려면  
분류 함수와 다운스트림(downstream) 수집기도 명시애햐 한다.

다운스트림 수집기: 해당 카테고리의 모든 원소를 담은 스트림으로부터 값을 생성하는 작업을 함

ex. 다운스트림 수집기에 toSet()

```java
words.collect(groupingBy(word -> alphabetize(word), toSet()));
```

이렇게 하면 리스트가 아닌 집합을 갖는 맵을 만듦.

ex. 다운스트림 수집기에 counting()

```java
Map<String, Long> freq = words
    .collect(groupingBy(String::toLowerCase, counting()));
```

각 카테고리와 해당 카테고리에 속하는 원소의 개수와 매핑된 맵을 얻는다.

<br>

---

groupingBy에 맵 팩토리를 넘겨 반환받을 Map의 구현체를 선택할 수도 있다.(HashMap, TreeMap, ConcurrentMap 등)

이것을 사용해서 `TreeMap<K, TreeSet<V>>` 같은 결과도 얻을 수 있다.

groupingByConcurrent 메서드는 groupingBy를 병렬 처리 가능한 버전으로 제공함.  
병렬 스트림에서 효율적으로 동작하고, 결과로 ConcurrentHashMap 반환

### 그 외 Collectors에 정의된 메서드

- partitioningBy
- summing~
- averaging~
- summarizing~
- reducing~
- filtering
- mapping
- flatMapping
- collectingAndThen
- maxBy
- minBy
- joining
