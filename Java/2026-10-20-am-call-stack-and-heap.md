# Call Stack과 Heap — 함수와 객체는 메모리에서 어떻게 관리될까?

- 날짜: 2026-10-20 (오전)
- 분야: Java / JavaScript / CS 기초
- 목표: 호출 스택과 힙의 역할, 객체 참조, 깊은 복사를 설명할 수 있다.

## 1. 문제

Java에서 `new User()`로 생성한 객체는 어디에 있을까? `const b = a`로 할당했는데 왜 a의 값도 변경될까? 함수가 여러 번 호출되면 어떤 순서로 실행되고 반환될까?

## 2. 핵심 개념

- **Call Stack(호출 스택)**: 함수 호출 시 실행 프레임을 쌓고 반환 시 제거한다. LIFO(후입선출) 방식이다.
- **Heap(힙)**: 객체 등 동적 데이터를 할당하는 메모리 영역이다.
- **참조**: 객체를 직접 복사하지 않고 같은 객체를 가리킬 수 있다.
- **GC(Garbage Collection)**: 더는 도달할 수 없는 객체 등을 회수한다. 회수 시점은 보장되지 않는다.

Java에서는 일반적인 `new` 객체가 힙에 할당되고 메서드의 지역 변수와 참조값 등은 스택 프레임에 놓인다. JavaScript는 엔진별 구현이 다르므로 정확한 물리적 배치를 단정하지 않고 호출 스택과 객체 참조의 개념을 이해하는 것이 중요하다.

## 3. 원인과 동작 원리

### 함수 호출 순서

```javascript
function first() { second(); }
function second() { third(); }
function third() { console.log('실행'); }
first();
```

호출: `first → second → third`, 반환: `third → second → first`.

### Java 객체 참조

```java
void test() {
    int age = 30;
    User user = new User("홍길동");
}
```

`age` 같은 지역 기본형 값과 `user` 참조값은 메서드 실행 프레임에서 관리되고, 일반적인 `User` 객체는 힙에 생성된다.

### JavaScript 참조 공유

```javascript
const original = { crclmNm: '교육과정A' };
const copy = original;
copy.crclmNm = '교육과정B';
console.log(original.crclmNm); // 교육과정B
```

`copy`와 `original`이 같은 객체를 가리켜 발생하는 현상이다.

## 4. 해결 방법과 실무 적용

데이터 원본을 독립적으로 보존하려면 깊은 복사가 필요하다.

```javascript
const original = { crclmNm: '교육과정A' };
const copy = JSON.parse(JSON.stringify(original));
copy.crclmNm = '교육과정B';
console.log(original.crclmNm); // 교육과정A
```

Vue3/Pinia에서 탭별 원본 데이터를 관리할 때도 적용한다.

```javascript
const saveOriginData = (tab) => {
  const data = getTabData(tab);
  if (data != null) originalData.value[tab] = JSON.parse(JSON.stringify(data));
};
```

`JSON.stringify` 방식은 Date, undefined, 함수, File 등의 데이터를 그대로 보존하지 못할 수 있다. `structuredClone`은 다양한 타입을 지원하지만 Vue 반응형 Proxy를 직접 복제하지 못한다.

## 5. 주의할 점

- Java의 StackOverflowError는 깊은 재귀 등으로 스택이 부족할 때 발생할 수 있다.
- Java의 OutOfMemoryError: Java heap space는 힙 할당 실패와 관련 있다.
- JavaScript 객체의 `===`는 구조적 값 비교가 아니라 동일 객체 참조인지 비교한다.
- `const b = a`는 객체 깊은 복사가 아니다.
- GC 동작과 정확한 메모리 배치는 언어 런타임 및 엔진 구현에 따라 달라진다.

## 6. 확인 문제

1. 일반적인 Java `new User()` 객체는 주로 어느 영역에 할당될까? → Heap
2. A → B → C 순서로 호출한 함수의 반환 순서는? → C → B → A
3. `const b = a`로 객체를 대입하면? → 동일한 객체를 참조한다.

## 7. 한 줄 정리

**Call Stack은 함수 실행 흐름을 관리하고 Heap은 객체 데이터를 관리한다. 참조 공유와 깊은 복사의 차이를 알아야 원본 데이터 변경 버그를 방지할 수 있다.**
