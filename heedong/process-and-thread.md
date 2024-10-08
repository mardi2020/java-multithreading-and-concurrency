# 프로세스와 스레드
- 멀티태스킹과 멀티프로세싱
- 프로세스와 스레드
- 스레드와 스케줄링
- 컨텍스트 스위칭

## 멀티태스킹과 멀티프로세싱
|    구분     |       관점        | 설명                                                                  |
|:---------:|:---------------:|---------------------------------------------------------------------|
|   멀티태스킹   | 소프트웨어<br>(운영체제) | 하나의 컴퓨터 시스템이 동시에 여러 작업을 수행<br>(CPU 사용 시간을 분할하여 동시에 작업이 수행하는 것처럼 보임) |
|  멀티프로세싱   |      하드웨어       | 둘 이상의 프로세서(여러 개의 CPU 코어)를 사용하여 여러 작업을 동시에 처리                        |

## 프로세스와 스레드
|  구분  | 설명                                                     |
|:----:|--------------------------------------------------------|
| 프로세스 | 실행중인 프로그램을 프로세스라고 함<br>(자바로 비유하면 클래스는 프로그램, 인스턴스는 프로세스) |
| 스레드  | 프로세스 내에서 실행되는 작업의 단위          |

<br>

**프로세스의 메모리 구성**
- **코드 섹션**: 실행할 프로그램의 코드가 저장되는 부분
- **데이터 섹션**: 전역 변수 및 정적 변수가 저장되는 부분(그림에서 기타에 포함)
- **힙 (Heap)**: 동적으로 할당되는 메모리 영역
- **스택 (Stack)**: 메서드(함수) 호출 시 생성되는 지역 변수와 반환 주소가 저장되는 영역(스레드에 포함)

> 그렇다면 JVM은 프로세스 구성 요소 중 어디로 올라가는가? → Heap 영역
- ref. https://dkswnkk.tistory.com/440

<br>

## 스레드와 스케줄링

작업(스레드)이 실행되기 위해서는 CPU를 할당받아서 연산을 실행해야 함
- 실제로 CPU에 의해 실행되는 단위는 스레드)
- 프로세스는 스레드들의 컨테이너 역할을 수행 (스레드들의 실행 환경을 제공)

이를 위해 OS는 스케줄링을 통해 스레드를 어떻게, 얼마나 오랫동안 CPU를 할당할지 결정 (CPU 시간을 여러 작업에 나누어 배분)


<br>

## 컨텍스트 스위칭

멀티태스킹을 위해 CPU가 여러 작업(스레드)을 번갈아가며 수행

스레드를 번갈아가며 호출하기 위해 CPU는 스레드의 상태를 저장하고 복원하는 작업을 수행

이러한 과정을 컨텍스트 스위칭이라고 함

- 멀티스레드는 대부분 효율적이지만, 컨텍스트 스위칭 과정이 필요하므로 항상 효율적인 것은 아님

<br>

### 최적의 CPU 수, 최적의 스레드 수

일반적으로 스레드가 하는 작업은 CPU 바운드 작업, I/O 바운드 작업으로 구분
- CPU 바운드 작업: CPU를 많이 사용하는 작업
  - 복잡한 수학 연산, 데이터 분석, 비디오 인코딩
- I/O 바운드 작업: I/O 작업(파일 읽기/쓰기, 네트워크 통신 등)이 많은 작업
  - 데이터베이스 쿼리 처리, 파일 읽기/쓰기, 네트워크 통신, 사용자 입력 처리

일반적으로 웹 애플리케이션 서버는 CPU 바운드 작업 보다는 I/O 바운드 작업이 많음

I/O 바운드 작업이 많다면 CPU를 거의 사용하지 않고 대기
- 따라서 CPU 코어 수에 맞게 스레드를 생성하면 CPU를 효율적으로 사용할 수 없음
- 이 때는 스레드 수를 증가시켜 CPU를 최대한 활용할 수 있도록 함

<br>

**CPU-바운드 작업**: CPU 코어 수 + 1개
- CPU를 거의 100% 사용하는 작업이므로 스레드를 CPU 숫자에 최적화

**I/O-바운드 작업**: CPU 코어 수 보다 많은 스레드를 생성
- CPU를 최대한 사용할 수 있는 숫자까지 스레드 생성 CPU를 많이 사용하지 않으므로 성능 테스트를 통해 CPU를 최대한 활용하는 숫자까지 스레드 생성
- 단 너무 많은 스레드를 생성하면 컨텍스트 스위칭 비용도 함께 증가 - 적절한 성능 테스트 필요