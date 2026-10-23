# Java Thread의 생성과 실행, Spring Boot 요청 처리

- 날짜: 2026-10-23 (오후)
- 분야: Java / 동시성
- 선수 학습: JVM 바이트코드 실행과 클래스 로딩 (2026-10-23 오전)

## 문제

오전에는 JVM이 Java 클래스를 로드하고 실행하는 과정을 배웠다. 그렇다면 Spring Boot 서버에 여러 HTTP 요청이 동시에 들어올 때 하나의 Java 메서드는 어떤 방식으로 실행될까?

예를 들어 `@Service`의 같은 메서드가 여러 요청에서 호출될 때, 각 요청의 지역 변수와 서비스 객체의 필드는 어떻게 다르게 취급될까?

**기대 결과:** 서로 다른 요청의 지역 데이터는 섞이지 않고, 공유 상태는 안전하게 관리된다.

**실제 위험:** 싱글턴 서비스의 변경 가능한 필드를 여러 요청이 동시에 수정하면 경쟁 상태(race condition)가 발생할 수 있다.

## 핵심 개념

### 1. 프로세스와 스레드

- **프로세스(Process)**: 실행 중인 프로그램의 인스턴스. JVM 프로세스는 힙 등 실행에 필요한 메모리 영역을 가진다.
- **스레드(Thread)**: 프로세스 안에서 코드를 실행하는 흐름. 하나의 JVM 프로세스에서 여러 Java 스레드가 동작할 수 있다.
- **스레드별 스택**: 각 스레드는 독립적인 JVM 스택을 갖고 메서드 호출 프레임과 지역 변수 등을 관리한다.
- **공유 힙**: 여러 스레드가 같은 힙 객체를 참조할 수 있다. 객체를 공유한다고 해서 자동으로 안전해지는 것은 아니다.

```text
JVM 프로세스
 ├─ Heap: CourseService 싱글턴 객체 (공유 가능)
 ├─ Thread A: JVM Stack A → 요청 A의 지역 변수
 └─ Thread B: JVM Stack B → 요청 B의 지역 변수
```

> 정확히는 객체의 필드가 힙 객체에 속하며, 지역 변수가 참조형이면 그 지역 변수에는 힙 객체를 가리키는 참조가 들어갈 수 있다. 따라서 지역 변수 자체가 스레드별이어도 그 참조가 가리키는 객체는 공유될 수 있다.

### 2. start()와 run()

- `thread.start()`: 새 스레드의 실행을 시작하도록 JVM에 요청한다. 새 스레드에서 `run()`이 수행된다.
- `thread.run()`: 일반 메서드 호출이다. 호출한 스레드에서 그대로 실행되며 새 스레드가 생성되지 않는다.
- `thread.join()`: 대상 스레드가 종료될 때까지 **호출한 스레드**가 기다린다.

### 3. Thread.State의 주요 상태

| 상태 | 의미 |
| --- | --- |
| `NEW` | 생성되었지만 `start()` 전 |
| `RUNNABLE` | JVM 관점에서 실행 가능하거나 실행 중 |
| `BLOCKED` | `synchronized` 모니터 락 획득 대기 |
| `WAITING` | 기한 없이 다른 동작을 기다림(예: `join()`) |
| `TIMED_WAITING` | 제한 시간 동안 대기(예: `sleep(1000)`) |
| `TERMINATED` | 실행 종료 |

이 상태들은 운영체제의 세부 스레드 상태와 1:1로 일치하지 않는다. `sleep()`은 스레드 실행을 잠시 지연시키지만 이미 가진 모니터 락을 자동으로 해제하지 않는다.

## 원인

Spring MVC의 전통적인 Servlet 기반 요청 처리는 일반적으로 서블릿 컨테이너의 작업 스레드에서 실행된다. 여러 요청이 동시에 같은 `@Service` 싱글턴 Bean을 호출할 수 있다.

문제가 되는 예시:

```java
@Service
public class CourseService {
    private int requestCount = 0; // 모든 요청이 공유하는 필드

    public int increase() {
        return ++requestCount;     // 읽기-수정-쓰기가 원자적이지 않음
    }
}
```

두 스레드가 같은 기존 값을 읽고 각각 증가시킬 수 있어, 두 번 호출해도 증가 결과가 하나만 반영되는 **lost update**가 발생할 수 있다. `volatile`만 붙여도 `++` 연산의 원자성은 보장되지 않는다.

## 해결 방법

### 1. start(), run(), join() 직접 확인

```java
public class ThreadDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("A: " + Thread.currentThread().getName());

        Thread worker = new Thread(() -> {
            System.out.println("B: " + Thread.currentThread().getName());
        }, "worker-1");

        worker.start(); // 새 스레드에서 B 실행
        worker.join();  // worker 종료까지 main 대기

        System.out.println("C: " + Thread.currentThread().getName());
    }
}
```

예상 출력:

```text
A: main
B: worker-1
C: main
```

`join()`이 있어 `B`가 `C`보다 먼저 출력된다. `worker.run()`으로 바꾸면 `B: main`이 출력된다.

### 2. Spring Boot에서는 공유 필드보다 지역 변수 사용

```java
@Service
@RequiredArgsConstructor
public class CourseService {
    private final CourseMapper courseMapper;

    public List<CourseDto> selectCourses(Long userId) {
        // 요청마다 실행되는 메서드의 지역 변수
        List<CourseDto> courses = courseMapper.selectByUserId(userId);
        return courses;
    }
}
```

`courses`라는 지역 변수는 호출마다 따로 존재한다. 단, 반환된 리스트나 그 내부 객체를 다른 스레드와 공유한다면 별도의 동시성 검토가 필요하다.

### 3. 공유 카운터가 정말 필요하다면

```java
@Service
public class RequestCounter {
    private final AtomicInteger count = new AtomicInteger();

    public int increase() {
        return count.incrementAndGet();
    }
}
```

`AtomicInteger`는 이 JVM 내부에서 카운터 증가를 원자적으로 처리한다. 하지만 서버가 여러 인스턴스로 실행된다면 인스턴스별 값이 분리된다. 전체 서버에서 일관된 카운트가 필요하면 DB나 Redis 등 별도 공유 저장소의 원자적 처리 방식을 검토한다.

### 4. 확인 문제 (짧은 실습)

다음 코드의 `B`는 어떤 스레드에서 실행될까? `A`, `B`, `C`의 출력 순서는?

```java
public class Quiz {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("A");
        Thread t = new Thread(
            () -> System.out.println("B: " + Thread.currentThread().getName()),
            "worker"
        );
        t.start();
        t.join();
        System.out.println("C");
    }
}
```

**정답:** `A → B: worker → C`.

**해설:** `start()`가 별도 스레드를 시작하고 `join()`이 해당 스레드의 종료까지 `main`을 대기시킨다. `join()`이 없다면 `B`와 `C`의 상대적 출력 순서는 보장되지 않는다.

**추가 실습:** `t.start()`를 `t.run()`으로 변경해 보고, 출력이 `B: main`으로 바뀌는 이유를 설명해 보자.

## 주의할 점

- `start()`를 호출했다고 스레드가 즉시 CPU에서 실행되는 것은 아니다. 스케줄링 순서는 보장되지 않는다.
- 같은 `Thread` 객체에 `start()`를 두 번 호출하면 `IllegalThreadStateException`이 발생한다.
- `Thread.sleep()`으로 실행 순서를 보장하려 하지 말자. 완료 대기는 `join()`, 비동기 결과는 `Future` 등 목적에 맞는 수단을 사용한다.
- Spring Bean의 `@Service`는 기본적으로 싱글턴 범위다. **싱글턴 자체가 스레드 안전성을 보장하지 않는다.**
- Spring MVC에서도 비동기 처리, 가상 스레드, 실행기 설정 등에 따라 실제 요청 실행 방식은 달라질 수 있다.
- `AtomicInteger`는 개별 증가 연산에는 적합하지만 여러 필드에 걸친 복합 비즈니스 트랜잭션을 해결하지 않는다.

## 한 줄 정리

> Java 스레드는 각자 실행 스택을 갖지만 힙 객체를 공유할 수 있으므로, Spring 싱글턴 서비스에서는 변경 가능한 공유 상태를 주의하고 `start()`와 `run()`을 구분해야 한다.

## 참고 자료

- Java Thread API: https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Thread.html
- Java Thread.State API: https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Thread.State.html
- Java AtomicInteger API: https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/AtomicInteger.html
