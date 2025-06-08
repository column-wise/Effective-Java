# item.09 try-finally 보다는 try-with-resources를 사용해라

자바 라이브러리에는 close 메서드를 이용해 직접 닫아줘야 하는 자원이 많다..  
하지만 자원 닫기는 놓치기 쉬워서 예측할 수 없는 성능 문제로 이어질 수 있다.

<br>

## try-finally

전통적으로 try-finally가 사용되었으나,

```java
// try-finally - No longer the best way to close resources!
static String firstLineOfFile(String path) throws IOException {
    BufferedReader br = new BufferedReader(new FileReader(path));
    try {
        return br.readLine();
    } finally {
        br.close();
    }
}
```

자원의 수가 늘어나면 지저분해진다

```java
// try-finally is ugly when used with more than one resource!
static void copy(String src, String dst) throws IOException {
    InputStream in = new FileInputStream(src);
    try {
        OutputStream out = new FileOutputStream(dst);
        try {
            byte[] buf = new byte[BUFFER_SIZE];
            int n;
            while ((n = in.read(buf)) >= 0)
                out.write(buf, 0, n);
        } finally {
            out.close();
        }
    } finally {
        in.close();
    }
}
```

문제가 또 있는데,

```java
BufferedReader br = new BufferedReader(new FileReader(path));
try {
    return br.readLine();      // ❗ 첫 번째 예외 발생 가능
} finally {
    br.close();                // ❗ 두 번째 예외 발생 가능
}
```

이런 경우에서 자바는 두 번째 예외가 첫 번째 예외를 덮어버려 첫 번째 원인이 로그에 남지 않음 -> 디버깅이 어려움

## try-with-resources

하지만 이러한 문제들은 try-with-resources 덕에 모두 해결되었다.  
이 구조를 사용하려면 해당 자원이 `AutoCloseable` 인터페이스를 구현해야 한다.

다음은 위 코드를 try-with-resources를 사용해 재작성한 코드이다.

```java
// try-with-resources - the best way to close resources!
static String firstLineOfFile(String path) throws IOException {
    try (BufferedReader br = new BufferedReader(
            new FileReader(path))) {
        return br.readLine();
    }
}
```

```java
// try-with-resources on multiple resources - short and sweet
static void copy(String src, String dst) throws IOException {
    try (InputStream in = new FileInputStream(src);
         OutputStream out = new FileOutputStream(dst)) {
        byte[] buf = new byte[BUFFER_SIZE];
        int n;
        while ((n = in.read(buf)) >= 0)
            out.write(buf, 0, n);
    }
}
```

catch 절도 사용할 수 있다.

```java
// try-with-resources with a catch clause
static String firstLineOfFile(String path, String defaultVal) {
    try (BufferedReader br = new BufferedReader(
            new FileReader(path))) {
        return br.readLine();
    } catch (IOException e) {
        return defaultVal;
    }
}
```

## 결론

try-with-resources 최고
