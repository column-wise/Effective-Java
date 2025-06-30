# item 45. 스트림은 주의해서 사용하라

## 스트림이란

스트림 API는 다량의 데이터 처리 작업(순차적 / 병렬적)을 돕고자 자바 8에 추가되었다.

이 API가 제공하는 추상 개념 중 핵심은 두 가지다.

1. 데이터 원소의 유한 혹은 무한 시퀀스
2. 파이프라인은 이 원소들로 수행하는 연산 단계를 표현

스트림 파이프라인은 소스 스트림에서 시작해 종단 연산(Terminal Operation)으로 끝나며, 그 사이에 하나 이상의 중간 연산(Intermediate Operation)이 있을 수 있다.

각 중간 연산은 한 스트림을 다른 스트림으로 변환

스트림 파이프라인은 지연 평가(Lazy Evaluation)된다.  
평가는 종단 연산이 호출될 때 이뤄지며, 종단 연산에 쓰이지 않는 데이터 원소는 계산에 쓰이지 않는다.

종단 연산이 없는 스트림 파이프라인은 아무 일도 하지 않으므로 빼먹지 말자.

스트림 API는 메서드 연쇄를 지원하는 플루언트 API(fluent API)다.

스트림을 제대로 사용하면 프로그램이 짧고 깔끔해지지만, 잘못 사용하면 읽기 어렵고 유지보수도 힘들어진다.

## 예시

다음 코드를 보자. 사전 파일에서 단어를 읽어 사용자가 지정한 문턱값보다 원소 수가 많은 아나그램(anagram) 그룹을 출력한다.

이 프로그램은 사용자가 명시한 사전 파일에서 각 단어를 읽어 맵에 저장한다.  
맵의 키는 그 단어를 구성하는 철자들을 알파벳 순으로 정렬한 값이고, 값은 같은 키를 공유한 단어들을 담은 집합이다.

그리고 values() 메서드로 아나그램 집합들을 얻어 원소 수가 문턱값보다 많은 집합들을 출력한다.

```java
// Prints all large anagram groups in a dictionary iteratively
public class Anagrams {
    public static void main(String[] args) throws IOException {
        File dictionary = new File(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        Map<String, Set<String>> groups = new HashMap<>();

        try (Scanner s = new Scanner(dictionary)) {
            while (s.hasNext()) {
                String word = s.next();
                groups.computeIfAbsent(alphabetize(word),
                    (unused) -> new TreeSet<>()).add(word);
            }
        }

        for (Set<String> group : groups.values())
            if (group.size() >= minGroupSize)
                System.out.println(group.size() + ": " + group);
    }

    private static String alphabetize(String s) {
        char[] a = s.toCharArray();
        Arrays.sort(a);
        return new String(a);
    }
}
```

다음 코드는 같은 일을 하지만 스트림을 과하게 사용한 예시이다.

```java
// Overuse of streams - don't do this!
public class Anagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);

        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(
                    groupingBy(word -> word.chars()
                            .sorted()
                            .collect(StringBuilder::new,
                                     (sb, c) -> sb.append((char) c),
                                     StringBuilder::append)
                            .toString()))
                 .values().stream()
                 .filter(group -> group.size() >= minGroupSize)
                 .map(group -> group.size() + ": " + group)
                 .forEach(System.out::println);
        }
    }
}
```

스트림을 과용하면 프로그램이 읽거나 유지보수하기 어려워진다.

절충안을 살펴보자.

```java
// Tasteful use of streams enhances clarity and conciseness
public class Anagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);

        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(groupingBy(word -> alphabetize(word)))
                 .values().stream()
                 .filter(group -> group.size() >= minGroupSize)
                 .forEach(g -> System.out.println(g.size() + ": " + g));
        }
    }

    // alphabetize method is the same as in original version
    private static String alphabetize(String s) {
        char[] a = s.toCharArray();
        Arrays.sort(a);
        return new String(a);
    }
}
```

alphabetize()를 별도 메서드로 분리: 가독성 증가, 테스트 용이  
스트림 체이닝은 최소화: 각 단계의 목적이 명확함  
의미 단위별 메서드 호출 순서: 그룹핑 → 필터링 → 출력 흐름이 직관적

alphabetize 메서드도 스트림을 이용해 구현할 수 있지만, 명확성이 떨어지고, 느려질 수도 있다.  
자바가 char용 스트림을 지원하지 않기 때문이다.

char 값들을 처리할 때는 스트림을 삼가는 편이 낫다.

## 스트림 VS 반복문

| 방식       | 표현 수단                                                        |
| ---------- | ---------------------------------------------------------------- |
| **반복문** | 코드 블록 (`for`, `while`, `if`, `break`, `return` 등)           |
| **스트림** | 함수 객체 (`lambda`, `method reference` 등) 기반 연산 파이프라인 |

---

| 반복문에서 가능                | 람다에서 불가능한 이유                                        |
| ------------------------------ | ------------------------------------------------------------- |
| 지역 변수 읽기/수정            | 람다는 **final 또는 effectively final** 변수만 접근 가능      |
| `return` 으로 외부 메서드 종료 | 람다는 외부 메서드에서 빠져나올 수 없음                       |
| `break` / `continue`           | 람다는 루프 제어 불가능                                       |
| **체크 예외** 던지기           | 람다는 checked exception을 던질 수 없음 (보통 try-catch 필요) |

## 스트림이 잘 맞는 작업들

| 작업 유형               | 예시                         |
| ----------------------- | ---------------------------- |
| 요소 일괄 변환          | `.map(x -> ...)`             |
| 필터링                  | `.filter(x -> 조건)`         |
| 합치기 (reduce)         | `.reduce(0, Integer::sum)`   |
| 그룹화 / 수집           | `.collect(groupingBy(...))`  |
| 조건 만족하는 요소 탐색 | `.anyMatch`, `.findFirst` 등 |

## 스트림이 처리하기 어려운 일

한 데이터가 파이프라인의 여러 단계를 통과할 때 이 데이터의 각 단계에서의 값들에 동시에 접근하기는 어렵다.

예를 들어  
스트림 파이프라인에서 .map()으로 값을 변환하면, 이전 값은 사라지고 새 값만 남음

```java
.map(p -> 2^p - 1) // Mersenne 수 계산
```

이후 .forEach(...) 에서는 p를 더 이상 사용할 수 없음

-> Pair 객체로 묶어 보관하면 p와 $2^p - 1$ 를 함께 보관할 수 있지만... 별로다.

```java
.map(p -> new Pair<>(p, 2^p - 1))
```

## 예시 2

1. 무한한 소수 스트림 생성

   ```java
   static Stream<BigInteger> primes() {
       return Stream.iterate(TWO, BigInteger::nextProbablePrime);
   }
   ```

   Stream.iterate(start, generator) 사용

2. 처음 20개의 메르센 소수 출력

   ```java
   public static void main(String[] args) {
       primes()
           .map(p -> TWO.pow(p.intValueExact()).subtract(ONE))         // 2^p - 1
           .filter(m -> m.isProbablePrime(50))                         // 소수 여부 확인
           .limit(20)
           .forEach(mp -> System.out.println(mp.bitLength() + ": " + mp)); // 역산해서 p도 출력
   }
   ```

두 예제에서 배울 점

| 내용                                      | 설명                                       |
| ----------------------------------------- | ------------------------------------------ |
| 스트림은 변환 과정에서 원래 값이 사라진다 | `map`을 쓰면 이전 값은 접근 불가           |
| 쌍(Pair) 객체는 쓰지 않는 것이 좋다       | 가독성과 간결함을 해친다                   |
| 가능한 경우 변환을 "역산"하라             | `bitLength()`로 p를 복구하는 방식처럼      |
| 스트림 메서드는 **복수형 이름**을 쓰자    | `primes()` → 가독성 향상 (스트림 API 관례) |

## 스트림과 반복문 중 어느쪽을 써야 할까

카드 덱 초기화 예제  
카드는 숫자와 무늬를 묶은 불변 값 클래스  
숫자와 무늬는 열거 타입

```java
// Iterative Cartesian product computation
private static List<Card> newDeck() {
    List<Card> result = new ArrayList<>();
    for (Suit suit : Suit.values())
        for (Rank rank : Rank.values())
            result.add(new Card(suit, rank));
    return result;
}
```

```java
// Stream-based Cartesian product computation
private static List<Card> newDeck() {
    return Stream.of(Suit.values())
        .flatMap(suit ->
            Stream.of(Rank.values())
                .map(rank -> new Card(suit, rank)))
        .collect(Collectors.toList());
}
```

| 방식   | 장점                                               | 단점                                     |
| ------ | -------------------------------------------------- | ---------------------------------------- |
| 반복문 | 직관적, 디버깅 쉬움, Java에 익숙한 개발자에게 친숙 | 코드가 길어짐                            |
| 스트림 | 간결하고 선언적, 함수형 스타일에 적합              | 흐름 파악이 어렵고 디버깅 어려울 수 있음 |

## 정리

| 조건                                                 | 적합한 방식 | 이유                       |
| ---------------------------------------------------- | ----------- | -------------------------- |
| 코드 흐름 제어가 필요함 (`return`, `break`, 예외 등) | **반복문**  | 람다는 불가능              |
| 로컬 변수에 값 누적 또는 상태 변경                   | **반복문**  | 람다는 로컬 변수 수정 불가 |
| 컬렉션의 변환, 필터링, 수집 등 일괄 처리             | **스트림**  | 선언적으로 표현 가능       |
| 루프 안 로직이 단순하고 순차적임                     | **스트림**  | 간결하고 가독성 좋음       |
