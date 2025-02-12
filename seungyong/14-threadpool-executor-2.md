# 섹션14. 스레드 풀과 Executor 프레임워크2

## ExecutorService 우아한 종료

가장 이상적인 종료

새로운 요청은 막고, 이미 진행중인 요청은 모두 완료한 다음 서버를 재시작

→ **graceful shutdown**

- shutdown() - 처리중인 작업이 없는 경우
    - shutdown() 호출 시 ExecutorService는 새로운 요청 거절
    - 스레드 풀 자원 정리
- shutdown() - 처리중인 작업이 있는 경우
    - shutdown() 호출 시 ExecutorService는 새로운 요청 거절
    - 스레드 풀의 스레드는 처리중인 작업 완료
    - 큐에 남아있는 작업도 모두 꺼내서 완료
    - 다 끝나면 자원 정리
- shutdownNow() - 처리중인 작업이 있는 경우
    - shutdown() 호출 시 ExecutorService는 새로운 요청 거절
    - 큐를 비우면서, 큐의 작업을 모두 꺼내 컬렉션으로 반환
    - 작업 중인 스레드에 인터럽트 발생
    - 자원 정리

## Executor 스레드 풀 관리

ExecutorService의 기본 구현체인 ThreadPoolExecutor의 속성

- corePoolSize: 스레드 풀의 기본 스레드 수
- maximumPoolSize: 스레드 풀에서 관리되는 최대 스레드 수
- keepAliveTime, TimeUnit unit: 기본 스레드 수를 초과해서 만들어진 스레드의 생존 가능 대기시간
    - 초과 시 제거됨
- BlockingQueue workQueue: 작업을 보관할 블로킹 큐

### 분석

- task1이 들어오면 Executor는 스레드 풀에 스레드가 core 사이즈 만큼 있는지 확인
    - 없으면 스레드 하나 생성
- core 사이즈 만큼 스레드가 이미 만들어져 있고, 스레드 풀에 스레드가 없으면 큐에 작업 보관
- 스레드 풀에 core 사이즈 만큼 이미 스레드가 있고, 큐도 가득 찬 경우
    - Executor는 maximumPoolSize 까지 초과 스레드를 만들어 작업 수행
    - 초과스레드 = max - core, max가 4고 core가 2면 초과 스레드를 2개 생성 가능
    - 생성된 초과 스레드는 방금 요청 온 작업을 먼저 수행(초과 하도록 만든 작업)
- 큐도 가득차고, 스레드풀의 스레드도 max 사이즈만큼 가득 찬 경우
    - `RejectedExecutionException` 발생

## Executor 전략

자바는 Executors 클래스를 통해 3가지 기본 전략을 제공

- newSingleThreadPool() : 단일 스레드 풀 전략
- newFixedThreadPool(nThreads): 고정 스레드 풀 전략
- newCachedThreadPool(): 캐시 스레드 풀 전략

### 고정 풀 전략

**newFixedThreadPool(nThreads)**

- 스레드 풀에 nThreads 만큼의 기본 스레드 생성, 초과 스레드 X
- 큐 사이즈에 제한 X
- 스레드 수가 고정되어 있으므로 자원 예측이 가능한 안정적인 방식

```java
new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS,
                                   new LinkedBlockingQueue<Runnable>())
```

> 점진적으로 요청이 많아져 작업의 처리 속도보다 큐에 쌓이는 속도가 더 빨라지면
자원은 여유 있지만, 사용자는 점점 느려지는 문제가 발생할 수 있다.
> 

### 캐시 풀 전략

**newCachedThreadPool()**

- 기본 스레드 사용 X, 60초 생존 주기를 가진 초과 스레드만 사용
- 초과 스레드 제한 X
- 큐에 작업 저장 X
- 모든 요청이 대기를 하지 않기 때문에 빠른 처리 가능 → 자원 최대로 사용 가능

```java
new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS,
                               new SynchronousQueue<Runnable>());
```

> SynchronousQueue란?
> 
> - BlockingQueue 인터페이스의 구현체 중 하나
> - 내부 저장 공간 X, 작업을 소비자 스레드에게 직접 전달
> - 생산자와 소비자를 동기화 하는 큐

서버가 자원을 최대한 사용하지만, 감당 가능한 임계점을 넘는 순간 시스템이 다운될 수 있다.

### 사용자 정의 풀 전략

전략을 세분화하여 사용하면 어느정도 상황에 대응할 수 있다.

- 일반: 안정적인 운영상태
- 긴급: 초과 스레드를 투입하여 처리하는 상태
- 거절: 대응이 힘들 경우 요청 거절

```java
ExecutorService es = new ThreadPoolExecutor(100, 200, 60, TimeUnit.SECONDS, new
ArrayBlockingQueue<>(1000));
```

- 100개의 기본 스레드
- 100개의 초과스레드, 60초 생명주기
- 1000개의 작업 큐

를 통해 1100개 까지 일반적인 대응, 1200개 까지 긴급 대응, 1201개 부터는 거절을 하는 전략을 짤 수 있다.

### 자주하는 실수

```java
 new ThreadPoolExecutor(100, 200, 60, TimeUnit.SECONDS, new
 LinkedBlockingQueue());
```

- 기본 스레드 100개
- 최대 스레드 200개
- 큐 사이즈: 무한대

큐가 다 차지 않기 때문에 초과 스레드가 생성하지 않는다

## Executor 예외 정책

ThreadPoolExecutor이 제공하는 예외 정책

- AbortPolicy: 새로운 작업 제출 시 `RejectedExcutionException` 발생, 기본 정책
- DiscardPolicy: 새로운 작업을 버림
- CallerRunsPolicy: 새로운 작업을 제출한 스레드가 대신 작업을 실행
- 사용자 정의(RejectedExecutionHandler): 개발자가 직접 거절 정책 정의