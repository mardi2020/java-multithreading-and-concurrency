# 섹션 13. 스레드 풀과 Executor 프레임워크1

## 스레드를 직접 사용할 때의 문제점

- 스드 생성 시간으로 인한 성능 문제
- 스레드 관리 문제
- `Runnable` 인터페이스의 불편함

### 1. 스레드 생성 비용으로 인한 성능 문제

스레드는 매우 무거움

- 메모리 할당: 각 스레드는 자신만의 호출 스택 보유, 즉 생성 시 메모리 할당을 해야함
- 운영체제 자원 사용: 스레드 생성은 운영체제 커널 수준에서 시스템 콜을 통해 이루어짐 → CPU, 메모리 자원 사용
- 운영체제 스케줄러 설정: 스레드가 새로 생성되면 스케줄러는 이 스레드를 관리 및 순서 조정을 해야함 → 오버헤드 발생

> 스레드 재사용을 하면 해당 문제를 해결할 수 있음
> 

### 2. 스레드 관리 문제

CPU와 메모리 자원은 한정되어 있음 → 스레드는 무한이 아님

시스템이 버틸 수 있는 최대 스레드의 수 까지만 스레드를 생성할 수 있도록 관리 필요

### 3. Runnable 인터페이스의 불편함

```java
public interface Runnable {
     void run();
}
```

- 반환 값이 없다: run() 메서드는 반환 값을 갖지 않음 → 실행 결과를 얻기 위해 별도 메커니즘 필요
- 예외 처리: run() 메서드는 체크 예외(checked exception)를 던질 수 없다. 메서드 내부에서 처리

### 해결

1, 2번 문제를 해결하기 위해서는 스레드를 생성, 관리 할 풀(Pool)이 필요

- 스레드를 관리하는 스레드 풀에 스레드를 미리 만듦
- 스레드는 스레드 풀에서 대기
- 작업 요청이 오면, 스레드 풀에서 스레드 하나를 조회
- 조회한 스레드로 작업 처리
- 작업이 완료되면 스레드 종료가 아닌 스레드 풀에 반환

스레드 풀을 사용하면 재사용이 가능해져 생성 시간 및 관리가 용이해짐

처리할 작업이 없으면 스레드는 `WAITING` 요청이 오면 `RUNNABLE` 

## Executor 프레임워크 소개

스레드 풀의 유지관리를 위해 스레드의 상태 전이 및 생산자 소비자 문제와 같은 문제를 해결해주는 프레임 워크 

개발자가 직접 스레드를 생성하고 관리하지 않고 효율적으로 처리하게 도와줌

### Executor 인터페이스

```java
package java.util.concurrent;
 public interface Executor {
     void execute(Runnable command);
}
```

### ExecutorService 인터페이스

```java
public interface ExecutorService extends Executor, AutoCloseable {
     <T> Future<T> submit(Callable<T> task);
     @Override
     default void close(){...}
		 ... 
}
```

- Executor 프레임워크를 사용할 때 대부분 이 인터페이스 사용

`ExecutorService` 인터페이스의 기본 구현체 → `ThreadPoolExecutor` 

### ThreadPoolExecutor 생성자

- corePoolSize: 스레드 풀에서 관리되는 기본 스레드의 수
- maximumPoolSize: 스레드 풀에서 관리되는 최대 스레드의 수
- keepAliveTime, TimeUnit unit: 기본 스레드 수를 초과해서 만들어진 스레드가 생존할 수 있는 대기시간, 초과시 제거
- BlockingQueue workQueue: 작업을 보관할 블로킹 큐

```java
12:10:54.451 [main] == 초기 상태 ==
12:10:54.461 [main] [pool=0, active=0, queuedTasks=0, completedTasks=0] main]==작업수행중==
12:10:54.461 [main] == 작업수행중 ==
12:10:54.461 [main] [pool=2, active=2, queuedTasks=2, completedTasks=0]
12:10:54.461 [pool-1-thread-1] taskA 시작 
12:10:54.461 [pool-1-thread-2] taskB 시작 
12:10:55.467 [pool-1-thread-1] taskA 완료 
12:10:55.467 [pool-1-thread-2] taskB 완료 
12:10:55.468 [pool-1-thread-1] taskC 시작 
12:10:55.468 [pool-1-thread-2] taskD 시작 
12:10:56.471 [pool-1-thread-2] taskD 완료 
12:10:56.474 [pool-1-thread-1] taskC 완료
12:10:57.465 [main] == 작업수행완료 ==
12:10:57.466 [main] [pool=2, active=0, queuedTasks=0, completedTasks=4] 
12:10:57.469 [main] == shutdown 완료 ==
12:10:57.468 [main] [pool=0, active=0, queuedTasks=0, completedTasks=4]
```

1. 초기 상태 시점에는 스레드 풀에 스레드를 미리 만들지 않음
2. 메인 스레드가 스레드 풀에 execute로 작업 호출
3. 작업 요청이 들어오면 처리하기 위해 스레드를 만든다.
4. 작업이 들어올 때마다 corePoolSize까지 스레드 생성
5. 작업이 완료되면 스레드 풀에 스레드 반납, `WAITING` 상태로 대기
6. 반납된 스레드는 재사용
7. close() 호출 시 ThreadPoolExecutor 종료, 스레드 풀의 스레드 제거

## Future

**Runnable**

```java
public interface Runnable {
	void run();
}
```

- Runnable의 run()은 반환 타입이 void → 값 반환 불가
- 예외가 선언되어 있지 않음 → 해당 인터페이스의 구현체는 모두 체크 예외를 던질 수 없음
    - 런타임은 제외

**Callable**

```java
public interface Callable<V> {
	V call() throws Exception;
}
```

- java.util.concurrent에서 제공
- Callable의 call()의 반환 타입은 제네릭 V → 값 반환 가능
- 예외가 선언 되어있으므로 체크 예외를 던질 수 있음

Callable은 다음과 같이 결과 값을 받을 수 있음

```java
Future<Integer> future = es.submit(new MyCallable());
Integer result = future.get();
```

그런데 MyCallable은 즉시 실행되어 결과를 반환하는 것이 불가능 함 → 다른 스레드에서 처리되기 때문

따라서 es.submit은 결과 대신 Future 객체를 반환

### Future 분석

- es.submit(taskA) 호출을 통해 taskA의 미래 결과를 알 수 있는 `Future` 객체 생성
- Future 객체 안에 taskA의 인스턴스 보관
    - 내부에 taskA의 작업 완료 여부, 결과 보관
- ThreadPoolExecutor의 블로킹 큐로 taskA가 아닌 Future 객체가 들어감
    - submit을 통해 작업을 전달할 때 생성된 Future은 즉시 반환됨
- 큐에 들어있던 Future을 꺼내 스레드 풀의 스레드가 작업 수행
    - FutureTask.run() → MyCallable.call()
- Future.get()을 통해 결과를 받을 수 있음
    - 완료 된 상태: 값을 즉시 반환
    - 미완료 상태: 요청한 스레드는 결과를 얻을 때 까지 대기(Blocking) → Thread.join()과 유사
        - Future을 처리하던 스레드가 요청한 스레드를 깨움

> Future를 사용하면 마치 멀티스레드를 사용하지 않고, 단일 스레드 상황에서 메서드를 호출하고 결과를 받는 것 같이 사용가능 + 예외도 던질 수 있음
> 

### Future 이유

- Future 없이 직접 반환: task1을 ExecutorService에 요청하고 결과 기다리고, task2를 요청하고 결과 기다리고 → 단일 스레드와 다른게 없음
- Future 반환: task1, task2를 각 스레드에 요청을 던져두고, 결과를 받을 때만 대기 → 대기시간을 공유하므로 절약할 수 있음

### Future 취소

cancel()의 매개변수에 따른 동작 차이

- cancel(true): Future를 취소 상태로 변경, 작업이 실행중이면 Thread.interrupt()를 호출해 작업 중단
- cancel(false): Future를 취소 상태로 변경, 이미 실행 중인 작업은 중단X

> 둘 다 취소는 되었기 때문에 값을 받을 수는 없음
> 

### Future 예외

요청스레드: es.submit(new ExCallable())을 호출하여 작업 전달

작업 스레드: ExCallable을 실행 → IllegalStateException 발생

- 작업 스레드는 Future에 예외를 담는다
- 예외 발생 → Future의 상태 `FAILED`

요청 스레드: future.get() 호출

- Future의 상태가 `FAILED` 면 ExecutionException 던짐
- 이 예외는 Future의 저장해둔 원본 예외 포함

> 마치 싱글 스레드의 일반적인 메서드를 호출하는 것과 같이 사용 가능
> 

## ExecutorService 작업 컬렉션 처리

- invokeAll(): 한번에 여러 작업 제출, 모든 작업 완료 까지 대기
- invokeAny(): 한번에 여러 작업 제출, 가장 먼저 완료된 작업 반환 나머지 작업은 인터럽트로 취소
