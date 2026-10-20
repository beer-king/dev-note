# Event Loop: Task와 Microtask의 실행 순서

- 학습 일자: 2026-10-20 (오후)
- 분야: JavaScript / Vue3
- 선수 학습: 오전 **Call Stack과 Heap**
- 목표: `setTimeout`, `Promise.then`, `queueMicrotask`, `nextTick`의 실행 시점을 예측한다.

## 문제

오전에 배운 Call Stack은 함수 호출 순서를 설명한다. 그런데 아래 코드는 `setTimeout`을 먼저 등록했는데도 왜 `Promise`가 먼저 출력될까?

```javascript
console.log('1. 시작');

setTimeout(() => console.log('2. 타이머'), 0);
Promise.resolve().then(() => console.log('3. Promise'));

console.log('4. 끝');
```

**실제 출력: `1. 시작 → 4. 끝 → 3. Promise → 2. 타이머`**

## 핵심 개념

- **Call Stack**: 현재 실행 중인 JavaScript 함수들이 쌓이는 공간. 동기 코드를 우선 실행한다.
- **Task(태스크, 흔히 macrotask라고도 부름)**: 스크립트 실행, 타이머 콜백, 사용자 이벤트 콜백 등 이벤트 루프가 처리하는 작업.
- **Microtask(마이크로태스크)**: `Promise.then/catch/finally`, `queueMicrotask`, `await` 뒤의 후속 실행 등.
- **Event Loop(이벤트 루프)**: 실행할 작업을 선택하고, 현재 작업이 끝나면 마이크로태스크 체크포인트를 수행하는 실행 모델.

브라우저 환경을 단순화하면 다음과 같다.

1. 현재 태스크에서 동기 코드를 끝까지 실행한다.
2. Call Stack이 비고 태스크가 끝나면 **대기 중인 마이크로태스크를 모두 실행**한다.
3. 마이크로태스크가 실행 중 새 마이크로태스크를 추가하면 그것도 같은 체크포인트에서 처리한다.
4. 브라우저가 필요에 따라 렌더링할 기회를 가진다.
5. 이후 다음 태스크를 실행한다.

> `setTimeout(fn, 0)`은 즉시 실행 명령이 아니다. 타이머 조건이 충족되면 이후 태스크로 실행할 기회를 얻는다. 정확한 지연 시간은 보장되지 않는다.

## 원인

`setTimeout` 콜백은 나중에 실행할 태스크로 예약된다. 반면 이미 이행된 Promise의 `then` 콜백은 마이크로태스크로 예약된다. 현재 동기 스크립트 실행이 끝나면 **다음 타이머 태스크에 앞서 마이크로태스크를 비우기 때문**에 Promise가 먼저 출력된다.

### 조금 더 어려운 예제

```javascript
console.log('A');

setTimeout(() => console.log('B'), 0);

Promise.resolve().then(() => {
  console.log('C');
  queueMicrotask(() => console.log('D'));
});

queueMicrotask(() => console.log('E'));

console.log('F');
```

**실행 결과: `A → F → C → E → D → B`**

| 순서 | 출력 | 이유 |
| --- | --- | --- |
| 1 | A | 동기 코드 |
| 2 | F | 동기 코드 |
| 3 | C | 먼저 예약된 Promise 마이크로태스크 |
| 4 | E | 기존에 예약된 다음 마이크로태스크 |
| 5 | D | C 실행 중 추가된 마이크로태스크 |
| 6 | B | 마이크로태스크가 모두 끝난 뒤 타이머 태스크 |

마이크로태스크는 실행 중 추가된 항목도 처리하기 때문에, 무한히 추가하면 렌더링이나 다음 태스크가 지연되는 **기아(starvation)** 현상이 발생할 수 있다.

## 해결 방법

### Vue3: 반응형 상태 변경 후 DOM 읽기

```vue
<script setup>
import { ref, nextTick } from 'vue';

const count = ref(0);
const label = ref(null);

async function updateAndRead() {
  count.value = 10;
  await nextTick();
  console.log(label.value?.textContent); // '10'
}
</script>

<template>
  <span ref="label">{{ count }}</span>
  <button @click="updateAndRead">변경</button>
</template>
```

Vue는 상태 변경에 따른 DOM 업데이트를 일반적으로 다음 틱에 묶어 처리한다. `await nextTick()`은 **Vue가 예약한 DOM 갱신이 완료된 뒤** 후속 코드를 실행하도록 한다.

단, `nextTick()`이 브라우저의 화면 페인트 완료나 **AG Grid 내부의 모든 비동기 렌더링 완료**를 보장하지는 않는다. AG Grid는 해당 API와 이벤트를 확인해야 한다.

### async/await와의 관계

```javascript
async function demo() {
  console.log('A');
  await Promise.resolve();
  console.log('C');
}

demo();
console.log('B');
// A → B → C
```

`await`는 해당 async 함수의 실행을 일시 중단하고, 기다린 Promise가 처리되면 후속 코드를 마이크로태스크로 이어간다. JavaScript 전체 스레드를 막는 것은 아니다.

### Java/Spring Boot와 비교

Java의 `CompletableFuture`, `ExecutorService`는 스레드/스레드풀을 이용한 비동기 처리 도구다. 브라우저 JavaScript의 이벤트 루프·마이크로태스크 큐와 **동일한 모델은 아니다**. 공통점은 '지금 실행할 코드'와 '나중에 실행할 후속 작업'을 구분해야 한다는 것이다.

## 실습 문제

다음 코드의 출력 순서를 **실행 전에** 예측해 보자.

```javascript
console.log(1);

setTimeout(() => console.log(2), 0);

Promise.resolve().then(() => {
  console.log(3);
  Promise.resolve().then(() => console.log(4));
});

console.log(5);
```

### 정답과 해설

**`1 → 5 → 3 → 4 → 2`**

동기 출력 `1, 5` → 첫 번째 Promise 후속 콜백 `3` → 그 안에서 추가한 마이크로태스크 `4` → 마지막으로 타이머 태스크 `2`.

## 주의할 점

- `setTimeout(fn, 0)`도 동기 코드와 마이크로태스크보다 먼저 실행되지 않는다.
- `await`는 일반적으로 async 함수의 나머지 부분을 뒤로 미루며, 외부의 동기 실행을 차단하지 않는다.
- 태스크 종류와 브라우저의 스케줄링에 따라 서로 다른 태스크 소스 간 순서를 무조건 FIFO로 가정해서는 안 된다.
- 렌더링은 매 반복마다 반드시 일어나지 않는다.
- 무거운 동기 연산은 Call Stack을 점유해 화면 응답을 막을 수 있다.

## 한 줄 정리

> **현재 동기 실행이 끝나면 마이크로태스크를 모두 처리한 다음, 다음 태스크를 실행한다.**

## 참고 자료

- [MDN — JavaScript execution model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model)
- [MDN — Using microtasks in JavaScript](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide)
- [Vue — nextTick()](https://vuejs.org/api/general.html#nexttick)
