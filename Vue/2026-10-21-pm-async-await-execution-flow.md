# async/await 실행 흐름: await은 무엇을 멈출까?

- 날짜: 2026-10-21 오후
- 분야: Vue3 / JavaScript
- 선수 학습: Promise, Event Loop

## 문제

Vue3에서 저장 API 실행 직후 목록을 조회하면 저장 전 데이터가 보일 수 있다.

```javascript
async function saveAndReload() {
  saveCourse();
  loadCourses();
}
```

`async`를 붙였지만 `saveCourse()`의 완료를 기다리지 않았기 때문이다.

## 핵심 개념

- `async` 함수는 항상 Promise를 반환한다.
- `await`은 현재 async 함수의 **나머지 실행**을 일시 중단한다. JavaScript 전체를 멈추지는 않는다.
- Promise가 이행되면 값을 받아 이어서 실행하고, 거부되면 예외처럼 처리한다.
- 중단된 함수는 마이크로태스크를 통해 다시 실행된다.

```javascript
async function getNumber() {
  return 10;
}
console.log(getNumber()); // Promise
console.log(await getNumber()); // 10 (async 함수나 모듈 최상위에서)
```

## 원인

```javascript
console.log('1');

async function run() {
  console.log('2');
  await Promise.resolve();
  console.log('3');
}

run();
console.log('4');
// 1 → 2 → 4 → 3
```

`run()`은 첫 `await` 이전까지 동기적으로 실행된다. `await` 이후 코드는 뒤에 재개되므로 `4`가 먼저 출력된다. `await 100`처럼 일반 값을 기다려도 이후 코드는 비동기적으로 재개된다.

## 해결 방법

### Vue3 / Pinia 실무: 저장 후 재조회

```javascript
const loading = ref(false);
const errorMessage = ref('');

async function saveAndReload() {
  loading.value = true;
  errorMessage.value = '';
  try {
    await courseStore.saveCourse();
    await courseStore.fetchCourses();
    console.log('저장 및 재조회 완료');
  } catch (error) {
    errorMessage.value = error.message ?? '처리 실패';
  } finally {
    loading.value = false;
  }
}
```

`saveCourse()`와 `fetchCourses()`가 실제 작업의 완료/실패를 반영하는 Promise를 반환해야 한다. 독립된 조회 작업이라면 `Promise.all([fetchCodes(), fetchCourses()])`로 함께 기다릴 수 있다.

### then()과 비교

```javascript
function withThen() {
  return Promise.resolve(5).then(n => n * 2);
}
async function withAwait() {
  const n = await Promise.resolve(5);
  return n * 2;
}
```

둘 다 결과 10으로 이행되는 Promise를 반환한다.

## 확인 문제

**문제 1.** 다음 출력 순서는?

```javascript
console.log('A');
async function test() {
  console.log('B');
  await 1;
  console.log('C');
}
test();
console.log('D');
```

**정답:** `A → B → D → C`. `await` 이후 코드는 마이크로태스크로 재개된다.

**문제 2.** 저장 API를 기다리려면 어떻게 고칠까?

```javascript
async function save() {
  store.saveCourse();
  console.log('저장 완료');
}
```

**정답:** `await store.saveCourse();`로 호출한다. 실패 처리에는 `try/catch`를 추가한다.

## 주의할 점

- `async`를 붙여도 내부 비동기 호출을 자동으로 기다리지는 않는다.
- `await`은 현재 함수만 일시 중단하며 다른 작업은 계속 실행될 수 있다.
- `fetch()`는 HTTP 404/500만으로 Promise를 거부하지 않으므로 `response.ok`를 검사한다.
- `forEach(async () => ...)`는 각 비동기 콜백 완료를 기다리지 않는다.
- 독립된 API 요청을 불필요하게 순차 처리하지 말자.

## 한 줄 정리

> `await`은 현재 async 함수의 이후 실행을 기다리게 하며, 함수 호출자는 Promise를 받는다.

## 참고 자료

- [MDN: async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)
- [MDN: await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await)
