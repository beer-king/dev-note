# JVM의 바이트코드 실행과 클래스 로딩

- 날짜: 2026-10-23 (오전)
- 분야: Java / JVM

## 문제

Spring Boot 프로젝트의 `CourseService.java`는 어떻게 실행될까? JVM이 Java 소스 코드를 직접 읽을까? `static` 블록은 언제 실행될까?

## 핵심 개념

```text
.java 소스 --javac--> .class 바이트코드
    --클래스 로더--> JVM --실행 엔진--> 실행
```

- **JDK**: `javac`, `javap` 같은 개발 도구와 실행 환경을 제공한다.
- **JVM**: 컴파일된 바이트코드를 실행한다.
- **클래스 로딩**: 필요한 클래스의 바이너리 데이터를 읽어 JVM에서 사용하도록 준비한다.
- **링크**: 검증, 준비, 해석 단계로 구성된다. 해석은 지연될 수 있다.
- **초기화**: 정적 필드 초기화식과 `static` 블록을 실행한다.
- **JIT**: 자주 실행되는 코드를 네이티브 코드로 최적화할 수 있다.

클래스 **로딩**과 **초기화**는 다르며 모든 클래스가 서버 시작과 동시에 초기화되는 것은 아니다.

## 원인

JVM 클래스 로딩과 Spring Bean 생성을 혼동하면 실행 흐름을 오해한다. JVM은 클래스 정보를 준비하고, Spring은 해당 클래스를 이용해 Bean을 생성하고 의존성을 주입한다. 또한 개발 환경에는 있던 클래스가 배포 환경의 실행 클래스패스에서 누락되면 오류가 발생할 수 있다.

## 해결 방법

### 1. 컴파일된 바이트코드 확인

`Hello.java`:

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println(add(10, 20));
    }
    static int add(int a, int b) {
        return a + b;
    }
}
```

```bash
javac Hello.java
java Hello
javap -c Hello
```

실행 결과는 `30`이다. `javap -c`로 `iadd`, `invokestatic` 같은 JVM 명령어를 확인할 수 있다.

### 2. 클래스 초기화 시점 확인

```java
class CourseConfig {
    static { System.out.println("초기화"); }
    static void print() { System.out.println("실행"); }
}
public class Main {
    public static void main(String[] args) {
        System.out.println("시작");
        CourseConfig.print();
        CourseConfig.print();
    }
}
```

출력:

```text
시작
초기화
실행
실행
```

첫 `CourseConfig.print()` 호출 때 클래스가 초기화되며, 두 번째 호출에서는 `static` 블록이 다시 실행되지 않는다.

### 3. Spring Boot 실무 적용

- `@Service`는 JVM의 클래스 로딩 명령이 아니라 Spring의 Bean 등록을 위한 컴포넌트 표식이다.
- 배포 시 클래스 관련 오류가 나면 JAR/WAR 내용, 런타임 의존성, JDK 버전, 최초 스택 트레이스를 확인한다.
- `NoClassDefFoundError`는 필요한 클래스 정의를 사용할 수 없을 때 발생하며, 클래스 초기화 실패 이후에도 나타날 수 있다.

### 확인 문제

```java
class A {
    static { System.out.println("A 초기화"); }
    static void run() { System.out.println("A 실행"); }
}
public class Main {
    public static void main(String[] args) {
        System.out.println("1");
        A.run();
        System.out.println("2");
        A.run();
    }
}
```

**정답:** `1 → A 초기화 → A 실행 → 2 → A 실행`

**해설:** 첫 `A.run()` 호출이 초기화를 유발하고 `static` 블록은 한 번만 실행된다.

## 주의할 점

- `ClassNotFoundException`은 명시적 클래스 검색 실패 등에서 발생하는 검사 예외다.
- `UnsupportedClassVersionError`는 실행 JVM이 해당 클래스 파일 버전을 지원하지 않을 때 발생할 수 있다.
- 컴파일 타임 상수에 대한 접근은 클래스 초기화를 유발하지 않을 수 있다.
- JVM의 클래스 로딩과 Spring의 Bean 생성은 구분해야 한다.

## 한 줄 정리

> Java 코드는 바이트코드로 컴파일되고 JVM이 클래스를 로드·링크·초기화해 실행하며, Spring은 그 위에서 Bean을 관리한다.

## 참고 자료

- JVM 명세: https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html
