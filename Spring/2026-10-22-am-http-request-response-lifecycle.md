# HTTP 요청·응답 흐름 — Vue3에서 Spring Boot까지

- 학습일: 2026-10-22 (오전)
- 분야: HTTP / Spring Boot / Vue3
- 선수 학습: Promise, async/await
- 학습 목표: 브라우저의 API 요청이 Spring Boot를 거쳐 JSON 응답으로 돌아오는 흐름을 설명하고 HTTP 상태 코드와 네트워크 오류를 구별한다.

## 문제

Vue3에서 `await fetch('/api/edu/courses')`로 교육과정 목록을 조회한다. 그런데 `await`은 **응답을 기다리는 방법**이지, 서버에서 요청이 처리되는 원리를 설명하지 않는다.

- `GET` 요청은 서버의 어디로 전달될까?
- `@RestController`가 반환한 Java 객체는 왜 브라우저에서 JSON으로 보일까?
- HTTP 404 응답인데 왜 `fetch()`의 `catch`가 실행되지 않을까?

## 핵심 개념

### 1. HTTP 요청과 응답

HTTP는 클라이언트와 서버가 요청(request)과 응답(response)을 주고받는 애플리케이션 계층 프로토콜이다.

| 구성 | 예시 | 의미 |
| --- | --- | --- |
| 요청 메서드 | `GET`, `POST` | 요청의 의도 |
| 경로 | `/api/edu/courses` | 요청 대상 리소스 |
| 헤더 | `Accept: application/json` | 수용할 응답 형식 |
| 요청 본문 | `{"courseName":"CS"}` | 서버에 전송하는 데이터 |
| 응답 상태 | `200 OK`, `404 Not Found` | 요청 처리 결과 |
| 응답 본문 | `[{"id":1,"name":"CS"}]` | 반환 데이터 |

HTTP는 **무상태(stateless)** 프로토콜이다. 로그인 상태 같은 사용자 맥락은 쿠키, 세션, 토큰 등의 별도 메커니즘으로 유지한다.

### 2. 상태 코드의 범주

- `2xx`: 요청 처리 성공 (예: `200 OK`, `201 Created`, `204 No Content`)
- `4xx`: 클라이언트 요청 관련 오류 (예: `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`)
- `5xx`: 서버 처리 오류 (예: `500 Internal Server Error`, `503 Service Unavailable`)

`401`은 인증이 필요하거나 유효하지 않음을, `403`은 접근이 거부됨을 나타낸다.

## 원인

### 1. 실제 처리 흐름

```text
Vue3 컴포넌트
  → Pinia action / fetch
  → 브라우저 HTTP 요청
  → 웹 서버·리버스 프록시(구성된 경우)
  → Spring Boot 내장 웹 서버(Tomcat 등)
  → Servlet Filter / Spring Security Filter Chain(구성된 경우)
  → DispatcherServlet
  → Controller → Service → MyBatis Mapper → DB
  ← Controller의 반환값
  ← HttpMessageConverter(Jackson을 이용한 JSON 직렬화)
  ← HTTP 상태·헤더·응답 본문
  ← 브라우저 fetch의 Response 객체
  ← response.json()으로 JavaScript 데이터 생성
```

- `DispatcherServlet`은 Spring MVC의 프런트 컨트롤러로 요청을 적절한 핸들러에 연결한다.
- `@RestController`의 반환값은 일반적으로 HTTP 응답 본문으로 작성된다.
- Spring Boot의 일반적인 JSON 설정에서는 Jackson 기반 `HttpMessageConverter`가 Java 객체를 JSON으로 직렬화한다.
- 브라우저의 `response.json()`은 응답 본문을 읽고 JSON 텍스트를 JavaScript 값으로 파싱하는 비동기 작업이다.
- 요청·응답의 실제 경로는 배포 구성, 필터, 인증 설정에 따라 달라질 수 있다.

### 2. Promise 성공과 HTTP 성공은 다르다

```javascript
const response = await fetch('/api/edu/courses');
console.log(response.status); // 404일 수도 있음
```

`fetch()`는 서버가 HTTP 404나 500으로 **응답한 경우에도** 일반적으로 `Response`를 반환하며 Promise는 이행된다. 네트워크 오류 등으로 응답을 얻지 못하면 거부될 수 있다.

## 해결 방법

### Vue3: 상태 코드 확인 후 JSON 읽기

```javascript
async function fetchCourses() {
  const response = await fetch('/api/edu/courses', {
    method: 'GET',
    headers: { Accept: 'application/json' }
  });

  if (!response.ok) {
    throw new Error(`교육과정 조회 실패: HTTP ${response.status}`);
  }

  return await response.json();
}
```

`response.ok`는 응답 상태 코드가 200~299이면 `true`다. `response.json()`은 Promise를 반환한다.

### Spring Boot: Controller → Service → Mapper

```java
@RestController
@RequestMapping("/api/edu")
@RequiredArgsConstructor
public class CourseController {
    private final CourseService courseService;

    @GetMapping("/courses")
    public ResponseEntity<List<CourseDto>> getCourses() {
        return ResponseEntity.ok(courseService.getCourses());
    }
}

@Service
@RequiredArgsConstructor
public class CourseService {
    private final CourseMapper courseMapper;

    public List<CourseDto> getCourses() {
        return courseMapper.selectCourses();
    }
}

@Mapper
public interface CourseMapper {
    List<CourseDto> selectCourses();
}

public record CourseDto(Long id, String name) {}
```

MyBatis XML 매핑과 DB 설정이 존재한다는 전제의 구조 예시다. `ResponseEntity.ok(...)`는 HTTP 200 응답을 생성하며, Spring의 메시지 컨버터가 DTO 목록을 JSON으로 직렬화한다.

### 확인하기: 브라우저 개발자 도구

1. 개발자 도구 → Network에서 요청 선택
2. Request URL, Method, Status 확인
3. Request Headers / Payload 확인
4. Response에서 실제 응답 본문 확인
5. 서버 로그에서 Controller 진입 여부 및 예외 확인

## 확인 문제

**문제 1.** `fetch('/api/edu/courses')`가 HTTP 404 응답을 받았다. `catch`가 자동으로 실행될까?

**정답:** 아니다. `fetch`는 404 응답만으로 거부되지 않는다. `response.ok`를 확인해 직접 오류를 던져야 한다.

**문제 2.** Spring Boot Controller에서 `List<CourseDto>`를 반환했는데 브라우저에는 JSON 배열이 왔다. 어떤 구성 요소가 변환할까?

**정답:** 일반적인 Spring Boot MVC 구성에서는 Jackson 기반 `HttpMessageConverter`가 직렬화한다.

**문제 3.** 다음 코드의 문제점은?

```javascript
const response = await fetch('/api/edu/courses');
const courses = response.json();
console.log(courses);
```

**정답:** `response.json()`도 Promise를 반환하므로 `courses`는 파싱된 배열이 아니라 Promise다. `const courses = await response.json();`으로 수정한다.

## 주의할 점

- `GET`은 일반적으로 조회, `POST`는 처리/생성에 사용한다. 실제 API 계약을 우선한다.
- `Content-Type`은 보내는 본문의 형식, `Accept`는 받고 싶은 응답 형식이다.
- `204 No Content`처럼 본문이 없는 응답에 `response.json()`을 호출하면 파싱 오류가 날 수 있다.
- `@RestController`의 자동 JSON 직렬화는 적절한 메시지 컨버터와 라이브러리 설정이 있다는 전제다.
- CORS는 브라우저의 교차 출처 요청 제한이며, 특정 요청은 실제 요청 전에 OPTIONS 사전 요청(preflight)이 발생한다.
- 서버가 `200`을 반환해도 업무 처리 결과가 성공이라는 보장은 없다. API의 응답 규약을 함께 확인한다.

## 한 줄 정리

> **Promise와 await은 응답을 기다리는 방식이고, HTTP는 요청·응답 규약이며, Spring MVC는 서버에서 그 요청을 처리하는 흐름이다.**

## 참고 자료

- [MDN — HTTP overview](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview)
- [MDN — HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)
- [MDN — Using the Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)
- [Spring Framework — DispatcherServlet](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet.html)
- [Spring Framework — HTTP Message Conversion](https://docs.spring.io/spring-framework/reference/web/webmvc/message-converters.html)
