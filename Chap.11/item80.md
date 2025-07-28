# item 80. 쓰레드보다는 실행자, 태스크, 스트림을 애용하라

## 실행자

`java.util.concurrent` 패키지의 등장으로, 실행자 프레임워크(Executor Framework)라고 하는 인터페이스 기반의 유연한 태스크 실행 기능을 담은 작업 큐를 단 한 줄로 생성할 수 있게 되었다.

```java
ExecutorService exec = Executors.new SingleThreadExecutor();  // Executor 생성
exec.execute(runnable);  // 실행할 태스크 전달
exec.shutdown();  // 실행자 종료
```

### 실행자 서비스의 주요 기능

- 특정 태스크가 완료되기를 기다린다.
- 태스크 모음 중 아무것 하나(`invokeAny`) 혹은 모든 태스크(`invokeAll`)가 완료되기를 기다린다.
- 실행자 서비스가 종료하기를 기다린다(`awaitTermination`).
- 완료된 태스크들의 결과를 차례로 받는다(`ExecutorCompletionService`)
- 태스크를 특정 시간에 혹은 주기적으로 실행하게 한다(`ScheduledThreadPoolExecutor`)

### 실행자의 생성

큐를 둘 이상의 쓰레드가 처리하게 하고 싶다면 간단히 다른 정적 팩토리를 이용하여 다른 종류의 실행자 서비스(쓰레드 풀)를 생성하면 된다.

우리가 원하는 실행자 대부분은 `java.util.concurrent.Executors`의 정적 팩토리들을 이용해 생성할 수 있다.

<details>
<summary> java.util.concurrent.Executors 에서 자주 쓰이는 Executor 종류 </summary>

| 메서드                                                                 | 생성되는 Executor 종류                | 특징 / 용도                                                                 |
| ---------------------------------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------- |
| **`newSingleThreadExecutor()`**                                        | `ExecutorService` (단일 쓰레드)       | 항상 **하나의 쓰레드**만 사용. 태스크가 순차적으로 실행됨.                  |
| **`newFixedThreadPool(int nThreads)`**                                 | `ExecutorService` (고정 쓰레드 풀)    | **고정된 수**의 쓰레드를 사용. 병렬 작업에 적절.                            |
| **`newCachedThreadPool()`**                                            | `ExecutorService` (캐시된 쓰레드 풀)  | 필요한 만큼 **무한히 쓰레드 생성**. 짧은 태스크, 서버엔 부적절.             |
| **`newSingleThreadScheduledExecutor()`**                               | `ScheduledExecutorService`            | 하나의 쓰레드로 **지연/주기적 실행** 수행.                                  |
| **`newScheduledThreadPool(int corePoolSize)`**                         | `ScheduledExecutorService`            | **여러 쓰레드로 주기적인 작업** 수행. cron-like 작업 가능.                  |
| **`newWorkStealingPool()`** (Java 8+)                                  | `ExecutorService` (ForkJoinPool 기반) | **코어 수만큼** 스레드를 갖는 **ForkJoinPool** 생성. 작업을 steal하며 처리. |
| **`unconfigurableExecutorService(ExecutorService)`**                   | Wrapper                               | 외부에서 설정 변경이 불가능한 Executor 래핑. API 제한용.                    |
| **`unconfigurableScheduledExecutorService(ScheduledExecutorService)`** | Wrapper                               | 위와 동일. 설정 고정된 스케줄러 제공.                                       |

</details>

### ThreadPoolExecutor

평범하지 않은 실행자를 원한다면 `ThreadPoolExecutor` 클래스를 직접 사용할 수도 있다.  
이 클래스로는 쓰레드 풀 동작을 결정하는 거의 모든 속성을 커스텀 할 수 있다.

### newCachedThreadPool

실행자 서비스를 사용하기에 까다로운 애플리케이션도 있다.  
작은 프로그램이나 가벼운 서버라면 `Executors.newCachedThreadPool`이 일반적으로 좋은 선택이다.  
특별히 설정할 게 없고, 일반적인 용도에 적합하게 동작한다.

하지만 `CachedThreadPool`은 무거운 프로덕션 서버에는 좋지 못하다.  
`CachedTrheadPool`에서는 요청받은 태스크들이 큐에 쌓이지 않고 즉시 쓰레드에 위임돼 실행된다.

가용한 쓰레드가 없다면 새로 하나를 생성한다.  
서버가 아주 무겁다면 CPU 이용률이 100%로 치닫고, 새로운 태스크가 도착하는 족족 또 다른 쓰레드를 생성하며 상황을 더욱 악화시킨다.

따라서 무거운 프로덕션 서버에서는 쓰레드 개수를 고정한 `Executors.newFixedThreadPool`을 선택하거나 완전히 통제할 수 있는 `ThreadPoolExecutor`를 직접 사용하는 편이 낫다.

## 태스크

작업 큐를 손수 만드는 일은 삼가야 하고, 쓰레드를 직접 다루는 것도 일반적으로 삼가야 한다.

쓰레드를 직접 다루면 Thread가 작업 단위와 수행 메커니즘 역할을 모두 수행하게 된다.  
반면 실행자 프레임워크에서는 작업 단위와 실행 메커니즘이 분리된다.

작업 단위를 나타내는 핵심 추상 개념이 태스크인데, 태스크에는 `Runnable`과 `Callable`이 있다.

`Callable`은 `Runnable`과 비슷하지만 값을 반환하고 예외를 던질 수 있다.

그리고 이 태스크를 수행하는 일반적인 메커니즘이 바로 실행자 서비스다.

## 포크-조인

자바 7이 되면서 실행자 프레임워크는 포크-조인(fork-join) 태스크를 지원하도록 확장되었다.

포크-조인 태스크는 포크-조인 풀이라는 특별한 실행자 서비스가 실행해준다.

`ForkJoinTask`의 인스턴스는 작은 하위 태스크로 나뉠 수 있고, `ForkJoinPool`을 구성하는 쓰레드들이 이 태스크를 처리하며, 일을 먼저 끝낸 쓰레드는 다른 쓰레드의 남은 태스크를 가져와 대신 처리할 수도 있다.

포크-조인 태스크를 직접 작성하고 튜닝하기란 어려운 일이지만, 포크-조인 풀을 이용해 만든 병렬 스트림을 이용하면 적은 노력으로 그 이점을 얻을 수 있다.

물론 포크-조인에 적합한 형태의 작업이어야 한다.
