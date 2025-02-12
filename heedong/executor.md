# 스레드 풀과 Executor 프레임워크

## 스레드를 직접 사용할 때의 문제점
1. 스레드 생성 시간으로 인한 성능 문제
   - 메모리 할당 (스레드 별 스택 메모리)
   - OS 자원/스케줄러 활용 (시스템 콜 및 스케줄링에 따른 오버헤드)
2. 스레드 관리 문제
   - 시스템이 유지될 수 있는 최대 스레드의 수 까지만 제어해야 함
3. `Runnable` 인터페이스의 불편함
   - 반환 값이 없고, 예외 처리가 불편함

<br> 

## Executor 프레임워크 
자바의 Executor 프레임워크는 멀티스레딩 및 병렬 처리를 쉽게 사용할 수 있도록 돕는 기능의 모음

`ExecutorService` 인터페이스의 기본 구현체는 `ThreadPoolExecutor`

```java
class ExecutorTest {
    
    private static final ExecutorService executor = Executors.newFixedThreadPool(10);
    
    void doSomething() {
        executor.submit(() -> {
            // task
        });
    }
}
```

`ExecutorService`는 `execute()` 메서드와 `submit()` 메서드 모두 호출이 가능
- `execute()` 메서드는 `Executor` 인터페이스의 구현
- `submit()` 메서드는 `ExecutorService` 인터페이스의 구현

차이점은 `execute()` 메서드는 `Runnable` 인터페이스를 구현한 객체만 인자로 받아 스레드 풀에 작업을 넣어줌

`submit()` 메서드는 `Runnable` 또는 `Callable` 인터페이스를 구현한 객체도 인자로 받아 스레드 풀에 작업을 넣어줌

또한 `submit()` 메서드는 `Future` 객체를 반환해줌 (`execute()` 메서드는 리턴 값을 반환하지 않음)
- `Future` 객체는 작업의 상태를 확인하거나 작업의 결과를 가져올 수 있음

<br>

### Runnable vs Callable
`Runnable` 인터페이스는 `void` 타입의 `run()` 메서드를 가지고 있음
```java
package java.lang;

@FunctionalInterface
public interface Runnable {
    void run();
}
```

`Callable` 인터페이스는 제네릭 타입 설정을 통해 리턴 값을 반환하는 `call()` 메서드를 가지고 있음

```java
package java.util.concurrent;

@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}

```

<br>

### Future
`Future` 는 작업의 미래 결과를 받을 수 있는 객체
- 비동기적인 작업의 결과를 나타내는 인터페이스
- `Future` 객체를 통해 작입이 진해될 때 요청 스레드는 대기하지 않고, 다른 작업을 수행할 수 있음

```java
class ExecutorTest {
    
    private static final ExecutorService executor = Executors.newFixedThreadPool(10);
    
    void doSomething() {
        var future = executor.submit(() -> {
            // task
        });
        
        val result = future.get();
    }
}
```

`future.get()` 을 호출했을 때
- **Future가 완료 상태**: `Future` 가 완료 상태면 `Future` 에 결과도 포함되어 있음
  - 이 경우 요청 스레드는 대기하지 않고, 값을 즉시 반환받을 수 있음
- **Future가 완료 상태가 아님**: `task` 가 아직 수행되지 않았거나 또는 수행 중
  - 이 때는 어쩔 수 없이 요청 스레드가 결과를 받기 위해 대기해야 함
  - 요청 스레드가 마치 락을 얻을 때처럼 결과를 얻기 위해 대기

<br>

### 종료 메서드
`void shutdown()`
- 새로운 작업을 받지 않고, 이미 제출된 작업을 모두 완료한 후에 종료
- 논 블로킹 동작 (이 메서드를 호출한 스레드는 대기하지 않고 즉시 다음 코드를 호출)


`List<Runnable> shutdownNow()`
- 실행 중인 작업을 중단하고, 대기 중인 작업을 반환하며 즉시 종료
- 실행 중인 작업을 중단하기 위해 인터럽트를 발생
- 논 블로킹 동작


`close()`
- 자바 19부터 지원하는 서비즈 종료 메서드
- `shutdown()` 을 호출하고, 하루(1일)를 기다려도 작업이 완료되지 않으면 `shutdownNow()` 를 호출
- 호출한 스레드에 인터럽트가 발생해도 `shutdownNow()` 를 호출한다.

```java
default void close() {
    boolean terminated = isTerminated();
    if (!terminated) {
        shutdown();
        boolean interrupted = false;
        while (!terminated) {
            try {
                terminated = awaitTermination(1L, TimeUnit.DAYS);
            } catch (InterruptedException e) {
                if (!interrupted) {
                    shutdownNow();
                    interrupted = true;
                }
            }
        }
        if (interrupted) {
            Thread.currentThread().interrupt();
        }
    }
}
```

<br>

## Executor 스레드 풀 관리
### 속성
```java
public ThreadPoolExecutor(
    int corePoolSize,
    int maximumPoolSize,
    long keepAliveTime,
    TimeUnit unit,
    BlockingQueue<Runnable> workQueue
) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, Executors.defaultThreadFactory(), defaultHandler);
}
```

`ExecutorService` 의 기본 구현체인 `ThreadPoolExecutor` 의 생성자는 아래 속성을 사용
- `corePoolSize` : 스레드 풀에서 관리되는 기본 스레드의 수
- `maximumPoolSize` : 스레드 풀에서 관리되는 최대 스레드 수
- `keepAliveTime` , `TimeUnit unit` : 기본 스레드 수를 초과해서 만들어진 초과 스레드가 생존할 수 있는 대기 시간, 이 시간 동안 처리할 작업이 없다면 초과 스레드는 제거
- `BlockingQueue workQueue` : 작업을 보관할 블로킹 큐

<br>

### Pool 생성 전략
- 고정 스레드 풀 전략
  - **newFixedThreadPool(nThreads)**
  - 트래픽이 일정하고, 시스템 안전성이 가장 중요
- 캐시 스레드 풀 전략
  - **newCachedThreadPool()**
  - 일반적인 성장하는 서비스
- 사용자 정의 풀 전략
  - **ThreadPoolExecutor(...)**
  - 다양한 상황에 대응

대부분의 서비스는 트래픽이 어느정도 예측 가능하므로 일반적인 상황이라면 고정 스레드 풀 전략이나, 캐시 스레드 풀 전략을 사용하면 충분