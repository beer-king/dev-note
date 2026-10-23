# Spring Boot의 Tomcat 요청 처리 스레드 풀

- 학습일: 2026-10-24 (오전)
- 분야: Spring Boot / Servlet / 동시성
- 선수 학습: JVM, Java Thread, Stack·Heap

## 문제

Vue3 화면에서 여러 사용자가 동시에 교육과정 조회 API를 호출한다. Spring Boot는 요청마다 새로운 Java `Thread`를 생성할까? 조회 SQL이 오래 걸리면 다른 사용자의 요청에는 어떤 영향이 있을까?

**기대:** 여러 HTTP 요청을 적절히 병행 처리한다.  
**실제 위험:** 요청 처리 스레드가 모두 장시간 점유되면 추가 요청의 응답이 지연되거나, 설정과 상황에 따라 거절될 수 있다.

## 핵심 개념

### 1. 요청 처리 스레드 풀(Thread Pool)

일반적인 **Spring Boot + 내장 Tomcat + Spring MVC(Servlet)** 구성에서는 Tomcat이 관리하는 작업 스레드가 HTTP 요청을 처리한다. 매 요청마다 새 스레드를 만들기보다 기존 스레드를 재사용한다.

```text
Vue3 / Axios
    ↓ HTTP 요청
Tomcat Connector
    ↓ 사용 가능한 작업 스레드 배정
http-nio-8080-exec-1
    ↓
Filter → DispatcherServlet → Controller → Service → Mapper → DB
    ↓
HTTP 응답
    ↓
스레드는 풀로 돌아가 다음 요청을 처리
```

- **Thread:** 코드를 실행하는 흐름.
- **Thread Pool:** 여러 작업에 재사용할 스레드를 관리하는 구조.
- **동시성:** 여러 요청이 겹치는 시간 동안 진행될 수 있음. CPU 코어 수와 작업 성격에 따라 실제 병렬 실행 여부는 다름.
- **요청 스레드:** Controller·Service를 호출하는 스레드. 기본적인 동기식 Spring MVC에서는 DB 응답을 기다리는 동안에도 점유될 수 있음.
- **Spring Bean:** 기본 싱글턴 Service는 여러 요청 스레드가 공유할 수 있음. 스레드별 지역 변수와 Bean의 인스턴스 필드를 구별해야 함.

### 2. 동작 원리

1. Tomcat이 클라이언트 연결과 HTTP 요청을 받아들인다.
2. 요청 처리 작업을 가용 작업 스레드가 실행한다.
3. 스레드가 Spring MVC의 Controller → Service → DB 호출을 수행한다.
4. 동기 JDBC/MyBatis 조회가 끝날 때까지 해당 요청 처리 스레드는 대기할 수 있다.
5. 응답을 완료한 스레드는 다른 요청 처리에 재사용된다.

**중요:** Vue3에서 `await axios.get(...)`을 사용해도 **서버의 요청 처리 스레드가 자동으로 해제되는 것은 아니다.** 프런트엔드의 비동기 대기와 서버의 동기식 요청 처리는 별개다.

## 원인

다음과 같은 코드가 있다면 요청 처리 스레드를 불필요하게 오래 점유한다.

```java
@RestController
@RequestMapping("/api/courses")
public class CourseController {

    @GetMapping("/slow")
    public String slow() throws InterruptedException {
        Thread.sleep(3000); // 학습용: 요청 스레드를 3초간 점유
        return Thread.currentThread().getName();
    }
}
```

`Thread.sleep(3000)` 동안 CPU를 계속 사용하는 것은 아니지만, **해당 요청을 처리하는 스레드는 다른 요청을 처리할 수 없다.** 실제 서비스에서는 느린 SQL, 외부 API 응답 대기, 락 대기 등으로 비슷한 병목이 발생할 수 있다.

스레드 풀의 처리 능력을 넘는 요청이 몰리면 대기 시간이 증가한다. 구체적인 대기·거절 동작은 Tomcat Connector, 연결 수, 큐, 타임아웃 설정에 따라 달라진다.

## 해결 방법

### 1. 실제 요청 스레드 이름 확인

```java
@RestController
@RequestMapping("/api/courses")
public class CourseController {

    @GetMapping("/thread")
    public Map<String, String> threadInfo() {
        return Map.of(
            "thread", Thread.currentThread().getName()
        );
    }
}
```

예시 응답:

```json
{"thread":"http-nio-8080-exec-1"}
```

여러 번 요청하면 같은 스레드 이름이 재사용될 수 있고, 동시 요청에서는 서로 다른 스레드가 사용될 수 있다. **정확한 이름은 서버 설정에 따라 다르다.**

### 2. 설정 위치 확인

Spring Boot 3.x에서 내장 Tomcat의 스레드 수는 다음과 같이 설정할 수 있다.

```yaml
server:
  tomcat:
    threads:
      max: 50
      min-spare: 10
```

- `max`: Tomcat 요청 처리 작업 스레드의 최대 개수.
- `min-spare`: 유지하려는 최소 유휴 스레드 수.

**무조건 max를 높이는 것이 해결책은 아니다.** DB 커넥션 풀(HikariCP), DB 처리량, CPU·메모리, 외부 API 한계도 함께 확인해야 한다.

### 3. 실무에서 우선 점검할 항목

1. 느린 SQL 실행 계획·인덱스·조회 건수 확인.
2. 외부 API 연결/읽기 타임아웃 설정.
3. HikariCP 커넥션 풀 고갈 및 장시간 트랜잭션 확인.
4. 요청 처리 시간과 Tomcat busy thread 지표 모니터링.
5. 장시간 작업은 필요 시 비동기 작업 처리 구조를 검토.

## 주의할 점

- 이 설명은 **전통적인 동기식 Spring MVC + Tomcat 플랫폼 스레드** 모델 기준이다. WebFlux, Servlet 비동기 처리, 가상 스레드 구성은 동작 방식이 다를 수 있다.
- `Thread.sleep()`은 실무에서 응답 지연을 구현하는 방법이 아니라 병목을 재현하는 학습용 예시다.
- Tomcat 스레드 풀과 **HikariCP DB 커넥션 풀은 서로 다른 자원**이다.
- 여러 스레드가 싱글턴 Service를 공유할 수 있으므로 요청별 상태를 변경 가능한 인스턴스 필드에 저장하지 않는다.

### 확인 문제

**Q1.** Tomcat 요청 처리 스레드가 최대 2개이고, 다른 제약이 없으며, 각 요청이 동기적으로 2초 걸린다면 요청 3개를 거의 동시에 보냈을 때 모두 완료되기까지 대략 얼마나 걸릴까?

**Q2.** Vue3에서 `await axios.get('/api/courses')`으로 호출하면 Spring Boot의 JDBC 조회 대기 중 Tomcat 요청 스레드가 자동 반환될까?

**정답 및 해설**

- **A1. 약 4초.** 첫 2개 요청이 약 2초에 완료되고, 남은 1개가 다음 2초에 완료되는 단순화된 가정이다. 실제 서버에는 큐·네트워크·DB 등 변수가 있다.
- **A2. 아니다.** 클라이언트의 `await`은 해당 JavaScript async 함수의 실행을 일시 중단할 뿐, 동기식 Servlet/JDBC 요청 스레드의 점유 상태를 바꾸지 않는다.

## 한 줄 정리

> Spring Boot의 동기식 MVC 요청은 Tomcat 스레드 풀의 스레드가 처리하며, 느린 DB·외부 API 호출은 요청 스레드를 오래 점유해 동시 처리량을 떨어뜨릴 수 있다.

## 참고 자료

- [Spring Boot — Embedded Web Servers](https://docs.spring.io/spring-boot/reference/web/servlet.html#web.servlet.embedded-container)
- [Spring Boot — Common Application Properties](https://docs.spring.io/spring-boot/appendix/application-properties/index.html)
- [Apache Tomcat — HTTP Connector](https://tomcat.apache.org/tomcat-10.1-doc/config/http.html)
