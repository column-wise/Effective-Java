# item 51. 메서드 시그니처를 신중히 설계하라

## 1. 메서드 이름을 신중히 짓자.
이해할 수 있고, 같은 패키지에 속한 다른 이름들과 일관되게 짓는 게 최우선이다.

개발자 커뮤니티에서 널리 받아들여지는 이름을 사용

긴 이름은 피하자.

## 2. 편의 메서드를 너무 많이 만들지 말자.

메서드가 너무 많은 클래스는 익히고, 사용하고, 문서화하고, 테스트하고, 유지보수하기 힘들다.

인터페이스도 마찬가지다. 이를 구현하는 사람과 사용하는 사람 모두 힘들다

## 3. 매개변수 목록은 짧게 유지하자.

4개 이하가 좋다.

같은 타입의 매개변수 여러 개가 연달아 나오는 경우가 특히 해롭다.

### 짧게 쪼개는 법

1. 여러 매서드로 쪼갠다.  
    ex. List API
    ```java
    // 긴 메서드가 없는 대신 조합해서 사용
    List<String> sub = list.subList(3, 10);     // sublist(시작, 끝)
    ```

    만약 이렇게 만들었다면????
    ```java
    int findIndexInSublist(List<String> list, int from, int to, String value);
    ```

    지저분하고 사용처도 제한적임.

<br>

2. 매개 변수 여러 개를 묶어주는 도우미 클래스를 만든다.
    ```java
    // 전: 매개변수 2개
    void playTurn(String rank, String suit);

    // 후: 카드 객체로 묶기
    class Card {
        String rank;
        String suit;

        Card(String rank, String suit) {
            this.rank = rank;
            this.suit = suit;
        }
    }

    void playTurn(Card card);  // ✅ 더 명확하고 단순함
    ```
<br>

3. 객체 생성에 사용한 빌더 패턴을 메서드 호출에 응용한다.

    ex. 리포트 생성기
    ```java
    class ReportBuilder {
        private LocalDate from;
        private LocalDate to;
        private String format;
        private boolean includeSummary;

        public ReportBuilder from(LocalDate from) {
            this.from = from;
            return this;
        }

        public ReportBuilder to(LocalDate to) {
            this.to = to;
            return this;
        }

        public ReportBuilder format(String format) {
            this.format = format;
            return this;
        }

        public ReportBuilder includeSummary(boolean flag) {
            this.includeSummary = flag;
            return this;
        }

        public Report generate() {
            // 유효성 검사 + 실행
            if (from == null || to == null) throw new IllegalStateException();
            return new Report(from, to, format, includeSummary);
        }
    }
    ```

    ```java
    Report report = new ReportBuilder()
    .from(LocalDate.of(2024, 1, 1))
    .to(LocalDate.of(2024, 12, 31))
    .format("pdf")
    .includeSummary(true)
    .generate();
    ```

## 4. 매개변수의 타입으로는 클래스보다는 인터페이스가 낫다.
예를 들어 메서드에 HashMap을 넘길 일은 전혀 없다.  
대신 Map을 사용하자.

그러면 HashMap, TreeMap, ConcurrentHashMap, 심지어 아직 존재하지 않는 Map도 넘길 수 있다.

## 5. boolean 보다는 원소 2개짜리 열거 타입이 낫다.

메서드 이름상 boolean이 더 명확한 경우를 제외하면  
열거 타입을 사용하는 것이 코드를 읽고 쓰기가 더 쉽고, 옵션을 추가하기 쉽다.

