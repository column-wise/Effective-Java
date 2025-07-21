# item 74. 메서드가 던지는 모든 예외를 문서화하라

메서드가 던지는 예외는 그 메서드를 올바르게 사용하는데 아주 중요한 정보다.  
따라서 메서드가 던지는 예외를 문서화하는 데 충분한 시간을 쏟아야 한다.

## 검사 예외
검사 예외는 항상 따로 선언하고, 각 예외가 발생하는 상황을 자바독의 `@throws` 태그를 활용하여 정확히 문서화하자.

ex. **good case**
```java
/**
 * 주어진 경로의 파일을 한 줄 읽습니다.
 *
 * @param path 파일 경로
 * @return 읽은 한 줄
 * @throws FileNotFoundException 파일이 존재하지 않는 경우
 * @throws IOException 파일을 읽는 도중 오류가 발생한 경우
 */
public String processFile(String path) throws FileNotFoundException, IOException {
    BufferedReader reader = new BufferedReader(new FileReader(path));
    String line = reader.readLine();
    reader.close();
    return line;
}
```

메서드가 던지는 각 예외를 뭉뚱그려 상위 클래스 하나로 선언하는 일은 피해야 한다.  

ex. **bad case**   
메서드가 던지는 모든 예외를 '이 메서드는 Exception을 던진다' 라고 선언하는 경우 -> XXXX

```java
public void processFile(String path) throws Exception {
    BufferedReader reader = new BufferedReader(new FileReader(path));
    String line = reader.readLine();
    reader.close();
}
```

이것은 메서드 사용자가 각 예외에 대처할 수 있는 힌트를 주지 못하고, 같은 맥락에서 발생할 여지가 있는 다른 예외들까지 삼켜버려 API 사용성을 크게 떨어뜨린다.

## 비검사 예외
비검사 예외도 검사 예외처럼 정성껏 문서화하면 좋다.

비검사 예외는 일반적으로 프로그래밍 오류를 뜻하는데, 일어날 수 있는 오류들이 무엇인지 알려주면,  
개발자는 자연스럽게 해당 오류가 나지 않도록 사용하게 된다.

인터페이스 명세의 경우 발생 가능한 비검사 예외를 문서로 남기는 일이 특히 중요하다.  
이 조건이 인터페이스의 일반 규약에 속하게 되어 그 인터페이스를 구현한 모든 구현체가 일관되게 동작하도록 해주기 때문이다.

다만 검사 예외는 `@throws` 태그로 문서화, 비검사 예외는 `@throws`를 사용하지 않는 것을 추천

검사 예외냐, 비검사 예외냐에 따라 API 사용자가 해야 할 일이 달라지기 때문에 구분해 주는 것이 좋다.

비 검사 예외를 모두 문서화하기 현실적으로 어려운 경우도 있다.  
클래스를 수정하여 새로운 비검사 예외를 던지게 되어도 소스 호환성과 바이너리 호환성이 그대로 유지되는 것이 가장 큰 이유다.

<details><summary>소스 호환성과 바이너리 호환성</summary>

🔹 1. 소스 호환성 (Source Compatibility)

기존에 작성된 소스 코드가 변경된 라이브러리/API와 함께 컴파일될 수 있는가?

예:
	•	외부 라이브러리에서 클래스의 메서드가 바뀌었을 때,
→ 우리가 기존 소스 코드를 다시 컴파일해도 컴파일 오류 없이 성공하면 소스 호환성이 유지된 것.

```java
// 우리가 작성한 코드
lib.doSomething();
```

```java
// 외부 라이브러리 변경: 새로운 비검사 예외 RuntimeException 추가
public void doSomething() throws RuntimeException { ... }
```

✅ 컴파일 성공함 → 소스 호환성 유지됨
왜냐면 RuntimeException은 **비검사 예외(Unchecked Exception)**이기 때문에
throws 절에 없더라도 컴파일러는 별로 신경 안 씀.

⸻

🔹 2. 바이너리 호환성 (Binary Compatibility)

이미 컴파일된 .class 파일이 변경된 라이브러리와 함께 정상적으로 실행될 수 있는가?

예:
	•	A.class가 외부 라이브러리의 B.class에 의존하고 있는데, B.class가 바뀌어도 A.class를 다시 컴파일하지 않고 그대로 실행 가능하면 바이너리 호환성이 유지된 것.

```java
// 이미 컴파일된 우리의 A.class는
b.doSomething();
```

```java
// 외부 라이브러리의 B.class는 수정됨:
public void doSomething() throws RuntimeException { ... }
```

✅ 우리의 .class 파일을 다시 컴파일하지 않아도 잘 실행됨 → 바이너리 호환성 유지됨
왜냐하면 RuntimeException은 비검사 예외라 JVM이 호출 시점에 체크하지 않음.

⸻

🔸 요약하면

| 항목              | 의미                                               | 예시                                                        |
|-------------------|----------------------------------------------------|--------------------------------------------------------------|
| 소스 호환성       | 기존 소스가 새 라이브러리와 컴파일되는지           | `throws RuntimeException`을 추가해도 소스는 컴파일 가능     |
| 바이너리 호환성   | 기존 `.class` 파일이 새 라이브러리와 실행되는지     | `.class` 파일 다시 컴파일하지 않아도 실행됨                  |



“비검사 예외는 문서화하기 어렵다”

	•	우리가 외부 라이브러리를 사용해 메서드를 만들고, 그 메서드가 어떤 예외를 던지는지 잘 문서화했다고 해도,
	•	나중에 외부 라이브러리가 새로운 비검사 예외를 추가하면,
	•	컴파일은 문제없이 되고 실행도 잘 되지만,
→ 우리는 그 새로운 예외를 문서에 쓰지 않았기 때문에 API 문서가 불완전해짐.


---


</details>

만약 다른 사람이 작성한 클래스를 사용하는 메서드에 대한 예외를 열심히 문서화했다고 해보자.  
후에 이 외부 클래스에 새로운 비검사 예외가 추가된다면, 우리 메서드는 문서에 언급되지 않은 새로운 비검사 예외를 전파하게 될 것이다.

\+ 같은 이유로 발생하는 예외들은 각각의 메서드가 아닌 클래스 설명에 추가해도 좋다