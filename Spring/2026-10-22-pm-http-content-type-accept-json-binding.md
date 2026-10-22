# HTTP Content-Type과 Accept, Spring MVC 데이터 바인딩

- 학습일: 2026-10-22 (오후)
- 분야: Spring Boot / Vue3 / HTTP
- 선수 학습: HTTP 요청·응답 흐름, Promise와 async/await
- 학습 목표: `Content-Type`과 `Accept`를 구분하고, `@RequestBody`·`@ModelAttribute`·`@RequestParam` 중 적절한 방식을 선택한다.

## 문제

Vue3에서 검색 조건을 Spring Boot로 보낼 때 다음 상황이 생긴다.

- `POST`로 JSON을 보냈는데 Controller의 `@ModelAttribute` DTO에 값이 들어오지 않는다.
- `Content-Type`을 변경했더니 `415 Unsupported Media Type` 오류가 발생한다.
- `consumes`와 `produces`를 무조건 설정해야 하는지 헷갈린다.

**핵심 질문:** HTTP 헤더는 어떤 역할을 하고, Spring MVC는 요청 데이터를 어떤 방식으로 Java 객체에 매핑할까?

## 핵심 개념

### 1. Content-Type과 Accept의 차이

| 헤더 | 의미 | 예시 |
| --- | --- | --- |
| `Content-Type` | **지금 보내는 본문**의 미디어 타입 | `Content-Type: application/json` |
| `Accept` | **응답으로 받고 싶은** 미디어 타입 | `Accept: application/json` |

예를 들어 요청 본문이 JSON이라면:

```http
POST /api/edu/courses/search HTTP/1.1
Content-Type: application/json
Accept: application/json

{"pageNo":1,"pageSize":20,"courseName":"CS"}
```

`Content-Type`은 요청 본문을 해석할 때, `Accept`는 응답 표현을 협상할 때 사용한다. 응답의 `Content-Type`은 **서버가 실제로 반환하는 본문 형식**을 나타낸다.

### 2. Spring MVC에서 요청 데이터를 받는 방법

| 방식 | 대표 입력 | 설명 |
| --- | --- | --- |
| `@RequestParam` | URL 쿼리 파라미터, 폼 필드 | 개별 파라미터 바인딩 |
| `@ModelAttribute` | 쿼리 파라미터, 폼 필드 | 여러 필드를 Java 객체로 바인딩 |
| `@RequestBody` | JSON/XML 등 요청 본문 | `HttpMessageConverter`를 통해 본문을 객체로 변환 |

`@ModelAttribute`는 JSON 요청 본문을 Jackson으로 역직렬화하는 용도가 **아니다**. JSON을 DTO로 받고 싶으면 일반적으로 `@RequestBody`를 사용한다.

## 원인

### 1. JSON과 폼 데이터의 처리 경로가 다르다

**JSON 본문**

```text
Vue fetch → HTTP body(JSON) → @RequestBody
→ MappingJackson2HttpMessageConverter → DTO
```

**쿼리 문자열 또는 폼 필드**

```text
Vue fetch → ?pageNo=1&pageSize=20
→ @RequestParam / @ModelAttribute → DTO 필드 바인딩
```

Spring MVC는 `@RequestBody`에서 본문과 `Content-Type`을 바탕으로 적절한 `HttpMessageConverter`를 선택한다. 일반적인 Spring Boot JSON 설정에서는 Jackson이 JSON을 Java 객체로 역직렬화한다.

### 2. consumes와 produces의 의미

```java
@PostMapping(
    value = "/courses/search",
    consumes = MediaType.APPLICATION_JSON_VALUE,
    produces = MediaType.APPLICATION_JSON_VALUE
)
```

- `consumes`: 매핑이 허용하는 **요청 본문 미디어 타입**을 제한한다.
- `produces`: 매핑이 생성할 **응답 미디어 타입**을 제한한다.
- 둘 다 **항상 필수는 아니다**. API 계약상 특정 형식으로 제한할 때 명시한다.
- 요청 `Content-Type`이 지원되지 않으면 보통 `415 Unsupported Media Type`, 수용 가능한 응답 형식을 만들 수 없으면 `406 Not Acceptable`이 발생할 수 있다.
- 잘못된 JSON 구문 또는 필요한 본문 누락 등은 일반적으로 `400 Bad Request`가 될 수 있다.

## 해결 방법

### 1. Vue3에서 JSON POST → Spring Boot @RequestBody

**Vue3 / Pinia action에서 사용할 수 있는 함수**

```javascript
async function searchCourses(params) {
  const response = await fetch('/api/edu/courses/search', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Accept: 'application/json'
    },
    body: JSON.stringify({
      pageNo: params.pageNo,
      pageSize: params.pageSize,
      courseName: params.courseName ?? ''
    })
  });

  if (!response.ok) {
    throw new Error(`조회 실패: HTTP ${response.status}`);
  }

  return await response.json();
}
```

**Spring Boot Controller와 요청 DTO**

```java
public record CourseSearchRequest(
    Integer pageNo,
    Integer pageSize,
    String courseName
) {}

@RestController
@RequestMapping("/api/edu")
@RequiredArgsConstructor
public class CourseController {
    private final CourseService courseService;

    @PostMapping("/courses/search")
    public ResponseEntity<List<CourseDto>> search(
            @RequestBody CourseSearchRequest request) {
        return ResponseEntity.ok(courseService.search(request));
    }
}
```

이 예시에서 `JSON.stringify`는 JavaScript 객체를 **JSON 문자열**로 직렬화하고, `@RequestBody`는 JSON을 DTO로 **역직렬화**한다. `CourseService`와 `CourseDto`는 프로젝트에 이미 정의되어 있다고 가정한다.

### 2. GET 검색 조건 → @RequestParam 또는 @ModelAttribute

**Vue3**

```javascript
const query = new URLSearchParams({
  pageNo: '1',
  pageSize: '20',
  courseName: 'CS'
});

const response = await fetch(`/api/edu/courses?${query}`, {
  headers: { Accept: 'application/json' }
});

if (!response.ok) throw new Error(`HTTP ${response.status}`);
const courses = await response.json();
```

**Spring Boot**

```java
@GetMapping("/courses")
public ResponseEntity<List<CourseDto>> getCourses(
        @RequestParam int pageNo,
        @RequestParam int pageSize,
        @RequestParam(defaultValue = "") String courseName) {
    return ResponseEntity.ok(
        courseService.search(pageNo, pageSize, courseName)
    );
}
```

검색 조건이 많다면 `@ModelAttribute`와 기본 생성자·setter를 갖춘 DTO를 사용할 수 있다.

```java
public class CourseQuery {
    private Integer pageNo;
    private Integer pageSize;
    private String courseName;

    public Integer getPageNo() { return pageNo; }
    public void setPageNo(Integer pageNo) { this.pageNo = pageNo; }
    public Integer getPageSize() { return pageSize; }
    public void setPageSize(Integer pageSize) { this.pageSize = pageSize; }
    public String getCourseName() { return courseName; }
    public void setCourseName(String courseName) { this.courseName = courseName; }
}

@GetMapping("/courses")
public ResponseEntity<List<CourseDto>> getCourses(
        @ModelAttribute CourseQuery query) {
    return ResponseEntity.ok(courseService.search(query));
}
```

두 `GET /courses` 메서드는 **대안 예시**이며 동일 Controller에 동시에 등록하면 매핑이 충돌하므로 하나만 선택한다.

### 3. 실습 문제

다음 요청을 받는 Controller를 완성해 보자.

```javascript
await fetch('/api/edu/courses/search', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ pageNo: 2, pageSize: 10 })
});
```

**문제 A.** `CourseSearchRequest` DTO로 본문을 받으려면 어떤 어노테이션을 써야 할까?

**정답:** `@RequestBody CourseSearchRequest request`. JSON 본문을 DTO로 변환하기 때문이다.

**문제 B.** 위 코드에서 `Content-Type` 대신 `Accept: application/json`만 설정하면 요청 본문 형식을 명확히 알릴 수 있을까?

**정답:** 아니다. `Accept`는 원하는 **응답 형식**이고, 요청 JSON 본문 형식은 `Content-Type`으로 알려야 한다.

**문제 C.** `@PostMapping(consumes = "application/json")`인데 클라이언트가 `Content-Type: text/plain`으로 전송한다면?

**정답:** 일반적으로 `415 Unsupported Media Type`이 발생한다.

## 주의할 점

- `@RequestBody`가 JSON을 자동 처리하려면 JSON 메시지 컨버터(Jackson 등)가 구성되어 있어야 한다.
- `Content-Type: application/json`을 붙이는 것만으로 객체가 JSON으로 변환되지는 않는다. `fetch`에서는 `JSON.stringify()`를 명시한다.
- `fetch`는 HTTP 4xx/5xx만으로 Promise를 거부하지 않으므로 `response.ok`를 확인한다.
- GET 요청에 JSON 본문을 억지로 넣지 않는다. `fetch`는 GET/HEAD 요청에 body를 허용하지 않는다.
- `FormData`로 파일을 업로드할 때는 브라우저가 multipart 경계(boundary)를 설정하도록 보통 `Content-Type`을 직접 지정하지 않는다.
- 외부 API가 `pageNo`·`pageSize`를 **문자열**로 요구한다면 그 API 계약을 따른다. 일반적인 Spring Boot DTO의 `Integer`와는 별개 문제다.
- JSON의 필드명과 DTO의 필드명, 자료형, null 허용 여부를 확인한다.
- `@ModelAttribute`는 폼 데이터/쿼리 파라미터에 적합하고, JSON 본문을 받는 방식과 혼동하지 않는다.

## 한 줄 정리

> **Content-Type은 보내는 본문 형식, Accept는 원하는 응답 형식이며, JSON 본문은 @RequestBody, 쿼리·폼 데이터는 @RequestParam 또는 @ModelAttribute로 받는다.**

## 참고 자료

- [MDN — Content-Type](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Type)
- [MDN — Accept](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Accept)
- [MDN — Using the Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)
- [Spring Framework — @RequestBody](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/requestbody.html)
- [Spring Framework — @ModelAttribute](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/modelattrib-method-args.html)
- [Spring Framework — HTTP Message Conversion](https://docs.spring.io/spring-framework/reference/web/webmvc/message-converters.html)
