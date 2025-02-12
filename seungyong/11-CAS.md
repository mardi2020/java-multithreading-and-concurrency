# 섹션 11. CAS - 동기화와 원자적 연산

## 원자적 연산

### 원자적 연산

해당 연산이 더 이상 나눌 수 없는 단위로 수행된다는 것을 의미 → 실행 or 실행 되지 않음

> 즉, 멀티스레드 상황에서 다른 스레드의 간섭 없이 안전하게 처리되는 연산
> 

원자적 연산이 아니지만 멀티스레드 상황에서 문제가 발생할 것 같은 경우, 앞서 배운 `synchronized` 블럭 or `Lock` 등을 사용해 임계 영역 설정

### Volatile

volatile는 여러 CPU 사이에 발생하는 캐시 메모리와 메인 메모리의 동기화 문제를 해결한다.

- 원자적 연산이 아니여서 발생하는 문제를 해결해주지 않음

### Synchronized

synchronized를 사용하면 안전하게 임계영역을 설정하여 블록 안의 연산을 원자적 연산으로 처리한다.

### AtomicInteger

`Synchronized` 블럭 안에서 증가, 감소 연산을 하듯이 AtomicInteger를 사용하면 원자적인 Integer를 사용할 수 있다.

> AtomicInteger, AtomicLong, AtomicBoolean과 같이 다양한 AtomicXxx 클래스 존재
> 

### 성능 비교

**일반적인 Integer를 사용한 증가연산**

- 가장 빠르다
- CPU 캐시 적극 활용
- 안전한 임계영역이 없고, volatile도 사용하지 않으므로 멀티스레드에서는 사용 불가

**Volatile를 사용한 증가연산**

- CPU 캐시를 사용하지 않고 메인 메모리 사용
- 임계 영역 사용 X → 멀티 스레드 환경에서 사용 불가
- 단일 스레드에선 일반 Integer보다 느림

**synchronized를 사용한 증가연산**

- `Synchronized` 를 사용하므로 멀티스레드 환경에서도 안전
- 조금 느리다

**AtomicInteger**

- 멀티스레드 상황에서 안전하게 사용 가능
- synchronized, Lock 사용 보다 성능이 빠름

> 왜?
> 

AtomicInteger는 synchronized와 Lock을 사용하지 않고 원자적 연산을 수행

## CAS 연산

### Lock 기반 방식의 문제점

락은 특정 자원을 보호하기 위해 스레드가 해당 자원에 대한 접근 제한

- 락 획득과 반납의 과정 반복 → 오버헤드

### CAS(Compare-And-Swap, Compare-And-Set)

- 락을 사용하지 않기 때문에 lock-free
- 락을 완전히 대체 X, 작은 단위의 일부 영역에 적용

<aside>
💡

**compareAndSet(0, 1)**

antomicInteger가 가지고 있는 값이 현재 0 이면, 이 값을 1로 변경하는 메서드

- 현재 값이 0 이면 1로 변경하고, true 반환 (성공 시)
- 현재 값이 0 이 아니면 변경 X, false 반환 (실패 시)

→ **이 메서드는 원자적으로 실행됨 (CPU 하드웨어 차원에서 제공하는 기능)**

</aside>

위 메서드는 두가지 과정으로 나눌 수 있는데

- x001의 값 확인
- 읽은 값이 0 이면 1로 변경

이 두 과정을 CPU가 원자적인 명령으로 만들기 위해 다른 스레드가 x001의 값을 변경하지 못하게 막음

## CAS 연산2

```java
private static int incrementAndGet(AtomicInteger atomicInteger) {
         int getValue;
         boolean result;
         do {
             getValue = atomicInteger.get();
             log("getValue: " + getValue);
             result = atomicInteger.compareAndSet(getValue, getValue + 1);
             log("result: " + result);
         } while (!result);
         return getValue + 1;
     }
```

- 위 코드처럼 value값을 읽고, 읽은 value값이 메모리 value값과 같은지 확인을 한 후 값을 증가
- 증가하는 로직인 CAS 연산이므로 멀티스레드 환경에서도 안전
    - CAS 성공 시 true 반환 후 do-while 탈출
    - CAS 실패 시 false 반환 후 do-while 재시작

## CAS 연산3

2개의 스레드로 CAS 연산2의 코드를 실행하면

```java
 start value = 0
 18:13:37.623 [ Thread-1] getValue: 0
 18:13:37.623 [ Thread-0] getValue: 0
 18:13:37.625 [ Thread-1] result: true
 18:13:37.625 [ Thread-0] result: false
 18:13:37.731 [ Thread-0] getValue: 1
 18:13:37.731 [ Thread-0] result: true
 AtomicInteger resultValue: 2
```

- Thread-0의 첫번 째 시도는 실패를 하여 getValue의 값이 증가하지 않은 것을 볼 수 있다.
    - Thread-1이 먼저 값을 올려 value의 값이 변경되었기 때문
    - false 반환 후 재시도
- Thread-0의 두번 째 시도는 성공하여 getValue의 값이 증가

### CAS 문제점

충돌이 빈번하게 발생하는 환경에서는 성능에 문제가 발생

- 여러 스레드가 자주 동시에 동일한 변수의 값을 변경하려 하는 경우

### CAS와 Lock 방식 비교

| Lock | CAS |
| --- | --- |
| 비관적 접근법 | 낙관적 접근법 |
| 데이터에 접근하기 전 항상 락 획득 | 락 사용X 바로 데이터 접근 |
| 다른 스레드의 접근 제한 | 충돌 발생 시 재시도 |
| 다른 스레드가 방해할 것이다 가정 | 대부분은 충돌이 없을것이다 가정 |

> 언제 CAS를 사용하는게 좋을 까?
> 

간단한 CPU 연산같이 빠르게 처리되면 충돌이 자주 발생하지 않으므로 빠른 처리가 가능한 연산

## CAS 락 구현

```java
public class SpinLock {
     private final AtomicBoolean lock = new AtomicBoolean(false);
		 public void lock() {
			 log("락 획득 시도");
			 while (!lock.compareAndSet(false, true)) {
				 // 락을 획득할 때 까지 스핀 대기(바쁜 대기) 한다.
					 log("락 획득 실패 - 스핀 대기"); 
				}
				log("락 획득 완료"); 
			}
     public void unlock() {
         lock.set(false);
				log("락 반납 완료"); 
			}
}
```

- CAS를 사용하지 않은 스핀락과 비교했을 때, 다음 두가지 연산이 원자적으로 묶여 임계영역이 뚫리는 일이 발생하지 않는다
    - 락 사용 여부 확인
    - 락의 값 변경

이런 방식의 락은 CPU가 `BLOCKED` 나 `WAITING` 으로 전이되지 않기 때문에 CPU 자원을 계속해서 사용하지만 그만큼 빠르게 락을 획득하고 실행할 수 있다.

- 임계 영역을 필요로 하지만 연산이 매우 짧을 경우 사용

## 정리

**CAS**

장점

- 낙관적 동기화: 락을 걸지 않고 안전하게 업데이트
    - 충돌이 자주 발생하지 않을 것을 가정, 충돌이 적은 환경에서 높은 성능
- 락 프리(Lock-Free): 락 사용 X → 락 획득 대기시간이 적음, 스레드 블로킹 X → 병렬 처리 효율적

단점

- 충돌이 빈번한 경우: 계속해서 재시도 해야하며, CPU 자원 지속적 소모 → 오버헤드 발생
- 스핀락과 유사한 오버헤드: 많은 충돌 시 많은 재시도 → 성능 저하

**동기화 락**

장점

- 충돌 관리: 락 사용을 통해 하나의 스레드만 리소스 접근 → 충돌 X
- 안정성: 복잡한 상황에서도 락을 통해 일관성 있게 동작
- 스레드 대기: 락을 대기하는 스레드는 CPU 사용 X

단점

- 락 획득 대기시간: 스레드가 락 획득을 위해 대기 → 소모 시간 발생
- 컨텍스트  스위칭 오버헤드: 락 획득 대기와 획득 시점에서 스레드의 상태가 변경 → 컨텍스트 스위칭 발생
    - 컨텍스트 스위칭으로 인한 오버헤드가 발생