# Spring @Transactional의 프록시 동작과 자기 호출(Self-invocation)

- 학습일: 2026-10-25 (오전)
- 분야: Spring Boot / AOP / 트랜잭션
- 선수 학습: JVM, Thread, Tomcat 요청 처리 스레드 풀
- 학습 목표: `@Transactional`이 실제로 적용되는 호출 경로를 이해하고, 일부 DB 저장만 남는 문제를 예방한다.

## 문제

Vue3에서 교육과정 신청 API를 호출하면 Spring Boot + MyBatis가 **신청 정보**와 **진도 초기 데이터**를 차례로 INSERT한다. 두 INSERT 중 두 번째에서 오류가 발생하면 둘 다 취소되어야 한다.

그런데 Service 메서드에 `@Transactional`을 붙였는데도 첫 번째 INSERT가 남는 경우가 있다.

**기대 결과:** 두 번째 INSERT 실패 시 첫 번째 INSERT도 롤백.  
**문제 상황:** 같은 클래스 안에서 트랜잭션 메서드를 직접 호출하여 Spring 트랜잭션 프록시를 거치지 않음.

## 핵심 개념

### 1. 트랜잭션(Transaction)

트랜잭션은 여러 DB 작업을 **하나의 작업 단위**로 묶는다.

- **COMMIT:** 모든 작업을 확정한다.
- **ROLLBACK:** 실패한 작업 단위의 변경을 취소한다.
- `@Transactional`은 Spring의 선언적 트랜잭션 설정이다. 애너테이션만으로 무조건 적용되는 것이 아니라 트랜잭션 관리 인프라와 호출 경로가 필요하다.

### 2. Spring 프록시(Proxy)는 무엇인가?

Spring의 기본 프록시 기반 트랜잭션 방식에서 다른 Bean이 트랜잭션 메서드를 호출하면 프록시가 호출을 가로챈다.

```text
Controller → Spring Proxy → 트랜잭션 시작(또는 기존 트랜잭션 참여)
                         → Service.save()
                         → MyBatis INSERT 1
                         → MyBatis INSERT 2
                         → 정상: COMMIT / 예외: 롤백 규칙에 따라 ROLLBACK
```

하지만 같은 객체 내부에서 `this.save()` 또는 `save()`를 호출하면 **프록시를 통과하지 않는다.**

```text
Controller → Service.outer() → this.save()
                               └─ 프록시를 거치지 않음
```

여기서 `save()`에 붙은 `@Transactional`은 해당 **내부 호출을 위해 새로 적용되지 않는다**. 단, `outer()`를 호출하기 전에 이미 활성화된 트랜잭션이 있다면 그 트랜잭션은 유지될 수 있다.

### 3. 기본 롤백 규칙

- `RuntimeException` 또는 `Error`: 기본적으로 롤백.
- 체크 예외(예: `IOException`): 기본적으로 자동 롤백 대상이 아님.
- 체크 예외도 롤백하려면 `@Transactional(rollbackFor = Exception.class)` 등을 사용한다.
- 기본 전파 옵션은 `Propagation.REQUIRED`: 기존 트랜잭션이 있으면 참여하고 없으면 새로 시작한다.

## 원인

다음은 문제가 발생할 수 있는 코드다.

```java
@Service
@RequiredArgsConstructor
public class EnrollmentService {
    private final EnrollmentMapper mapper;

    public void enroll(Long courseId, Long userId) {
        saveEnrollment(courseId, userId); // 같은 객체의 메서드 직접 호출
    }

    @Transactional
    public void saveEnrollment(Long courseId, Long userId) {
        mapper.insertEnrollment(courseId, userId);
        mapper.insertProgress(courseId, userId);
    }
}
```

Controller가 `enroll()`을 호출하면, 외부에서 프록시를 통해 들어왔더라도 `enroll()` 자체에는 트랜잭션 설정이 없다. 이어서 호출한 `saveEnrollment()`는 **자기 호출**이므로 해당 메서드의 `@Transactional`이 적용되지 않는다.

별도 트랜잭션이 없는 일반적인 MyBatis-Spring/JDBC 환경에서는 INSERT가 각각 커밋되어 부분 저장 문제가 발생할 수 있다.

## 해결 방법

### 1. 가장 단순한 방법: 외부에서 호출하는 메서드에 적용

```java
@Service
@RequiredArgsConstructor
public class EnrollmentService {
    private final EnrollmentMapper mapper;

    @Transactional
    public void enroll(Long courseId, Long userId) {
        mapper.insertEnrollment(courseId, userId);
        mapper.insertProgress(courseId, userId);
    }
}
```

Controller 등 **다른 Bean**이 `enroll()`을 호출하면 트랜잭션 프록시가 적용된다.

### 2. 역할을 분리해야 한다면 Bean을 나눈다

```java
@Service
@RequiredArgsConstructor
public class EnrollmentFacade {
    private final EnrollmentTxService enrollmentTxService;

    public void enroll(Long courseId, Long userId) {
        enrollmentTxService.saveEnrollment(courseId, userId);
    }
}

@Service
@RequiredArgsConstructor
public class EnrollmentTxService {
    private final EnrollmentMapper mapper;

    @Transactional
    public void saveEnrollment(Long courseId, Long userId) {
        mapper.insertEnrollment(courseId, userId);
        mapper.insertProgress(courseId, userId);
    }
}
```

`EnrollmentFacade` → **다른 Spring Bean의 프록시** → `saveEnrollment()` 순서이므로 트랜잭션이 적용된다.

### 3. 실제 트랜잭션 적용 여부 확인

```java
import org.springframework.transaction.support.TransactionSynchronizationManager;

System.out.println(
    TransactionSynchronizationManager.isActualTransactionActive()
);
```

별도 트랜잭션이 없는 상태에서 자기 호출한 메서드 내부에서는 일반적으로 `false`, 프록시를 통해 트랜잭션이 시작된 메서드 내부에서는 `true`를 확인할 수 있다.

또한 아래 로깅 설정으로 트랜잭션 경계 관련 로그를 확인할 수 있다.

```yaml
logging:
  level:
    org.springframework.transaction.interceptor: TRACE
```

### 4. 실무 연결: DB 커넥션 점유

Tomcat 요청 스레드가 Service를 실행하고, 트랜잭션 처리 중 MyBatis가 JDBC 커넥션을 사용한다. 일반적인 JDBC 트랜잭션에서는 획득한 커넥션이 트랜잭션 종료까지 묶일 수 있다.

따라서 트랜잭션 안에서 느린 외부 API를 호출하거나 불필요하게 오래 대기하면 **HikariCP 커넥션 부족**으로 이어질 수 있다. 트랜잭션 경계는 필요한 DB 작업 중심으로 짧게 잡는다.

## 주의할 점

- `@Transactional`은 **기본 프록시 모드**에서 외부 호출을 가로챈다. AspectJ 방식은 동작이 다르다.
- 같은 클래스 내부 호출뿐 아니라 `new EnrollmentService(...)`로 직접 만든 객체도 Spring 프록시가 아니다.
- `private` 메서드에 애너테이션을 붙여도 기본 프록시 방식으로 해당 메서드를 직접 가로챌 수 없다. 학습 단계에서는 **다른 Bean에서 호출하는 public 메서드**를 기준으로 생각한다.
- 예외를 메서드 내부에서 잡고 정상 반환하면 기본적으로 프록시가 실패를 알 수 없어 커밋할 수 있다.
- 여러 DB나 외부 API까지 하나의 로컬 JDBC 트랜잭션으로 자동 롤백되는 것은 아니다.

### 확인 문제

다음 코드에서 `save()`의 `@Transactional`은 새 트랜잭션을 시작할까? (호출 전 활성 트랜잭션이 없고, 기본 프록시 방식이라고 가정)

```java
@Service
public class CourseService {

    public void execute() {
        save();
    }

    @Transactional
    public void save() {
        System.out.println(
            TransactionSynchronizationManager.isActualTransactionActive()
        );
    }
}
```

**선택지:** ① true ② false ③ 컴파일 오류

**정답: ② false.** `execute()`에서 `save()`를 직접 호출해 프록시를 우회한다. 따라서 `save()`의 `@Transactional`로 트랜잭션이 시작되지 않는다.

**짧은 실습:** `execute()`에 `@Transactional`을 추가하고 다른 Bean에서 `execute()`를 호출하면 출력은 어떻게 바뀔까? → **true**. 이번에는 `execute()` 진입 시 프록시가 트랜잭션을 시작하기 때문이다.

## 한 줄 정리

> Spring의 기본 `@Transactional`은 프록시를 통과한 호출에 적용되므로, 같은 클래스 내부의 자기 호출은 새 트랜잭션 적용을 우회할 수 있다.

## 참고 자료

- [Spring Framework — Using @Transactional](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
- [Spring Framework — Rolling Back a Declarative Transaction](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/rolling-back.html)
- [Spring Framework — Understanding AOP Proxies](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)
