# 동시성 컬렉션

`java.util` 패키지에 소속되어 있는 컬렉션 프레임워크는 스레드 세이프(Thread Safe)하지 않음
- Thread Safe: 여러 스레드가 동시에 접근해도 괜찮은 경우

```java
public class SimpleListMainV0 {

    public static void main(String[] args) {
        addAll(new BasicList());
    }

    private static void test(SimpleList list) throws InterruptedException {
        log(list.getClass().getSimpleName());

        // A를 리스트에 저장하는 코드
        Runnable addA = new Runnable() {
            @Override
            public void run() {
                list.add("A");
                log("Thread-1: list.add(A)");
            }
        };

        // B를 리스트에 저장하는 코드
        Runnable addB = new Runnable() {
            @Override
            public void run() {
                list.add("B");
                log("Thread-2: list.add(B)");
            }
        };

        Thread thread1 = new Thread(addA, "Thread-1");
        Thread thread2 = new Thread(addB, "Thread-2");
        thread1.start();
        thread2.start();
        thread1.join();
        thread2.join();
        log(list); // actual: [B, null]
    }
}
```

따라서 컬렉션에서도 여러 스레드에서 동시에 접근한다면 스레드 세이프한 컬렉션을 사용해야 함

<br>

### 방법 1. 프록시 패턴 활용
```java
public static <T> List<T> synchronizedList(List<T> list) {
    return new SynchronizedRandomAccessList<>(list);
}
```

컬렉션 내부의 모든 메서드를 `synchronized` 키워드를 추가하여 관리하는 것은 유지보수에 어려움이 있음

따라서 프록시 패턴을 활용해서 컬렉션을 감싸는 방법을 사용 (e.g. `Collections.synchronizedList()`)
- 내부의 모든 메서드에 `synchronized` 키워드가 추가

하지만 단점도 존재함
- 동기화 오버헤드 발생
  - `synchronized` 키워드가 멀티스레드 환경에서 안전한 접근을 보장함
  - 다만 각 메서드 호출 시마다 동기화 비용이 추가됨, 이로 인해 성능 저하가 발생할 수 있음
- 전체 컬렉션에 대해 동기화가 이루어지기 때문에 잠금 범위가 넓어질 수 있음
  - 이는 잠금 경합(lock contention)을 증가시키고, 병렬 처리의 효율성을 저하시키는 요인 유발
  - 모든 메서드에 대해 동기화를 적용하다 보면 특정 스레드가 컬렉션을 사용하고 있을 때 다른 스레드들이 대기해야 하는 상황이 빈번해질 수 있음
- 정교한 동기화가 불가능함
  - `synchronized` 프록시를 사용하면 컬렉션 전체에 대한 동기화가 이루어지지만 특정 부분이나 메서드에 대해 선택적으로 동기화를 적용하는 것은 어려움
  - 이는 과도한 동기화로 이어질 수 있음

따라서 이 방식은 동기화에 대한 최적화가 이루어지지 않는 구현
- 자바는 이런 단점을 보완하기 위해 `java.util.concurrent` 패키지에 동시성 컬렉션(concurrent collection)을 제공

<br> 

### 방법 2. 동시성 컬렉션 사용
자바 1.5부터 `java.util.concurrent` 패키지에는 고성능 멀티스레드 환경을 지원하는 다양한 동시성 컬렉션 클래스들을 제공

#### 컬렉션 인터페이스

| 컬렉션 인터페이스 | 동시성 컬렉션 클래스 |           설명            |
|:------------------:|:--------------:|:-----------------------:|
| `List`             | `CopyOnWriteArrayList`  |     `ArrayList`의 대안     | 
| `Set`              | `CopyOnWriteArraySet`  |      `HashSet`의 대안      |  
| `Set`              | `ConcurrentSkipListSet` |      `TreeSet`의 대안      | 
| `Map`              | `ConcurrentHashMap`   |      `HashMap`의 대안      |   
| `Map`              | `ConcurrentSkipListMap` |      `TreeMap`의 대안      | 

`LinkedHashSet` , `LinkedHashMap` 처럼 입력 순서를 유지하면서 멀티스레드 환경에서 사용할 수 있는 `Set` , `Map` 구현체는 제공하지 않음
- 필요하다면 `Collections.synchronizedXxx()` 를 사용해야 함
- 설계 철학, 성능 이슈(내부 추가 동기화) 및 코드 복잡성 등을 고려하여 제공하지 않음

<br>

#### BlockingQueue 인터페이스

 | 동시성 컬렉션 클래스 | 설명                                                                      |
|:------------------:|:------------------------------------------------------------------------|
| `ArrayBlockingQueue` | - 크기가 고정된 블로킹 큐<br>- 공정(fair) 모드를 사용할 수 있음, 다만 공정(fair) 모드를 사용 시 성능이 저하 |
| `LinkedBlockingQueue` | - 크기가 무한하거나 고정된 블로킹 큐<br>- `ArrayBlockingQueue` 보다 더 효율적인 메모리 사용 |
| `PriorityBlockingQueue` | - 우선순위 큐<br>- 우선순위가 높은 요소를 먼저 처리하는 블로킹 큐 |
| `SynchronousQueue` | - 데이터를 저장하지 않는 블로킹 큐<br>- 생산자가 데이터를 추가하면 소비자가 그 데이터를 받을 때까지 대기 |
| `DelayQueue` | - 지연된 요소를 처리하는 블로킹 큐<br>- 각 요소는 지정된 지연 시간이 지난 후에야 소비될 수 있음 |

<br>

## 동시성 컬렉션이 모든 상황에서 효율적일까?
> No. 동시성 컬렉션이 모든 상황에서 최적화된 silver bullet은 아님

Thread Safe한 ArrayList 를 기준으로 비교해보자
- SynchronizedList VS CopyOnWriteArrayList

<br>

#### 결론
자료의 크기나 작업의 종류에 따라 가장 효율적인 컬렉션은 다룰 수 있음
- `SynchronizedList`: 쓰기 작업이 읽기 작업보다 많은 경우
- `CopyOnWriteList`: 읽기 작업이 쓰기 작업보다 많은 경우
