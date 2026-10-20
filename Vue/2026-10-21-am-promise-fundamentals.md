# Promise의 상태와 체이닝 — 비동기 결과를 어떻게 전달할까?

- 학습일: 2026-10-21 (오전)
- 분야: JavaScript / Vue3
- 선수 학습: Call Stack과 Heap(10/20 오전), Event Loop와 Task/Microtask(10/20 오후)
- 학습 목표: Promise의 세 상태, `then/catch/finally`의 실행 방식, 체이닝과 오류 전파를 설명할 수 있다.

## 문제

Vue3에서 API를 호출한 다음 결과를 처리하는 코드를 보자.

```javascript
const result = fetch('/api/courses');
console.log(result); // Response 데이터가 아니라 Promise 객체
```

왜 `fetch()`의 반환값을 곧바로 응답 데이터처럼 사용할 수 없을까? 실패했을 때 `catch()`는 어디에서 실행될까?

## 핵심 개념

**Promise**는 미래에 완료될 비동기 작업의 **결과(또는 실패 이유)를 나타내는 객체**다.

| 상태 | 의미 | 다음 상태 |
| --- | --- | --- |
| `pending` | 아직 완료되지 않음 | `fulfilled` 또는 `rejected` |
| `fulfilled` | 성공적으로 완료되어 결과값을 가짐 | 변경 불가 |
| `rejected` | 실패하여 이유(error)를 가짐 | 변경 불가 |

`fulfilled`와 `rejected`를 합쳐 **settled(완료된 상태)**라고 한다. 한 번 완료되면 상태와 결과는 바뀌지 않는다.

- `.then(onFulfilled)`: 성공한 값을 전달받는다.
- `.catch(onRejected)`: 앞선 체인에서 거부된 이유나 던져진 예외를 처리한다.
- `.finally(onFinally)`: 성공·실패와 관계없이 실행한다. 인자를 받지 않으며 일반적으로 기존 결과를 그대로 통과시킨다.
- `then/catch/finally`는 **각각 새로운 Promise를 반환**하므로 체이닝할 수 있다.

## 원인

### 1. Promise는 결과값 자체가 아니다

```javascript
const promise = Promise.resolve({ id: 10, title: 'CS 기초' });
console.log(promise instanceof Promise); // true

promise.then(course => {
  console.log(course.title); // CS 기초
});
```

`Promise.resolve(...)`는 이미 이행된 Promise를 만든다. 그래도 `then` 콜백은 동기 코드 중간에 즉시 실행되지 않고 **마이크로태스크**로 실행된다.

```javascript
console.log('A');
Promise.resolve('B').then(value => console.log(value));
console.log('C');
// A → C → B
```

### 2. 체이닝은 이전 단계의 반환값을 다음 단계로 전달한다

```javascript
Promise.resolve(10)
  .then(num => num * 2)          // 20을 이행값으로 갖는 새 Promise
  .then(num => {
    console.log(num);            // 20
    return num + 5;
  })
  .then(num => console.log(num)); // 25
```

`then`에서 일반 값을 반환하면 다음 Promise는 그 값으로 이행된다. Promise를 반환하면 그 Promise의 최종 상태를 따르며, `throw`하면 다음 Promise는 거부된다.

**주의:** `then` 콜백에 `return`이 없으면 다음 `then`에는 `undefined`가 전달된다.

### 3. 오류는 처리할 때까지 다음 체인으로 전파된다

```javascript
Promise.resolve('시작')
  .then(() => {
    throw new Error('API 처리 실패');
  })
  .then(() => console.log('실행되지 않음'))
  .catch(error => {
    console.log(error.message); // API 처리 실패
    return '기본값';
  })
  .then(value => console.log(value)) // 기본값
  .finally(() => console.log('정리 완료'));
```

`catch`에서 값을 반환하면 체인이 정상 이행 상태로 복구될 수 있다. 반대로 다시 `throw`하면 거부 상태가 계속 전파된다.

## 해결 방법

### Vue3 API 호출에서 Promise 처리하기

아래는 `fetch`를 이용한 독립적인 예제다.

```javascript
import { ref } from 'vue';

const courses = ref([]);
const loading = ref(false);
const errorMessage = ref('');

function loadCourses() {
  loading.value = true;
  errorMessage.value = '';

  return fetch('/api/courses')
    .then(response => {
      // fetch는 HTTP 404/500만으로 Promise를 reject하지 않는다.
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      return response.json(); // Promise를 반환: 다음 then이 JSON 결과를 받음
    })
    .then(data => {
      courses.value = data;
    })
    .catch(error => {
      errorMessage.value = error.message;
      // 호출자에게 실패를 전파해야 한다면 여기서 다시 throw
    })
    .finally(() => {
      loading.value = false;
    });
}
```

**실무 포인트:** 로딩 표시를 종료하는 코드는 성공·실패 양쪽에서 실행되어야 하므로 `finally`에 배치하는 것이 적합하다. `fetch`의 네트워크 실패와 HTTP 오류 응답은 구별해야 한다.

## 확인 문제

**문제 1.** 출력 순서를 예측하자.

```javascript
console.log(1);
Promise.resolve(2)
  .then(n => n + 1)
  .then(n => console.log(n));
console.log(4);
```

정답: `1 → 4 → 3`. 동기 로그가 먼저 출력되고 `then` 체인이 마이크로태스크로 이어진다.

**문제 2.** 다음 코드에서 두 번째 `then`이 받는 값은?

```javascript
Promise.resolve(5)
  .then(n => { n * 2; })
  .then(value => console.log(value));
```

정답: `undefined`. 중괄호를 쓴 콜백에서 `return`을 생략했기 때문이다.

**문제 3.** `catch`에서 `return []`을 하면 그 다음 `then`은 실행될까?

정답: 실행된다. `catch`가 정상적으로 반환한 값 `[]`을 다음 `then`이 받는다.

## 주의할 점

- Promise가 **fulfilled**라고 해서 HTTP 요청의 비즈니스 처리가 성공했다는 뜻은 아니다. `fetch`는 HTTP 404/500 응답에도 일반적으로 이행된다.
- `finally`는 이전 결과값을 전달받지 않는다. 다만 `finally` 안에서 예외를 던지거나 거부된 Promise를 반환하면 체인의 결과가 바뀔 수 있다.
- `catch`가 에러를 처리하고 반환하면 호출자는 성공으로 인식할 수 있다. 에러 전파가 필요하면 다시 던져야 한다.
- Promise는 **비동기 결과를 표현**할 뿐, 콜백 코드가 자동으로 별도 스레드에서 실행된다는 의미는 아니다.
- `async/await`는 Promise를 다루는 다른 문법이다. 다음 학습에서 다룬다.

## 한 줄 정리

> **Promise는 미래의 결과를 나타내며, `then`은 값을 연결하고 `catch`는 오류를 처리하며 `finally`는 공통 정리를 수행한다.**

## 참고 자료

- [MDN — Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)
- [MDN — Using promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises)
- [MDN — Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)
