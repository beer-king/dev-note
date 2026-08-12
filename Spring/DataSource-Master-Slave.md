# Spring DataSource와 Master/Slave 구조

## 문제

Spring 애플리케이션에서 데이터베이스 연결 설정을 보다가 다음 내용이 궁금했다.

- `DataSource`는 어떤 역할을 하는가?
- JDBC URL과 JDBC Driver는 무엇인가?
- Connection Pool과 HikariCP는 왜 필요한가?
- Master/Slave 데이터베이스를 나누는 이유는 무엇인가?
- `@Transactional(readOnly = true)`를 붙이면 자동으로 Slave DB를 사용하는가?

## 핵심 개념

### DataSource

Spring에서 `DataSource`는 애플리케이션이 데이터베이스 연결을 얻기 위한 설정과 진입점을 제공하는 객체다.

> DataSource = 애플리케이션이 DB에 접속하고 Connection을 얻기 위한 설정 및 진입점

일반적인 설정은 다음과 같다.

```yaml
spring:
  datasource:
    url: jdbc:mariadb://localhost:3306/lms
    username: root
    password: password
    driver-class-name: org.mariadb.jdbc.Driver
```

| 설정 | 역할 |
| --- | --- |
| `url` | 접속할 DB 서버와 데이터베이스 정보 |
| `username` | DB 접속 계정 |
| `password` | DB 접속 비밀번호 |
| `driver-class-name` | 사용할 JDBC Driver |

### JDBC URL

다음 URL은 MariaDB 서버의 `lms` 데이터베이스에 접속한다는 뜻이다.

```text
jdbc:mariadb://192.168.0.10:3306/lms
│     │          │          │    │
│     │          │          │    └─ Database 이름
│     │          │          └────── Port
│     │          └───────────────── DB 서버 주소
│     └──────────────────────────── DB 종류
└────────────────────────────────── JDBC
```

- DBMS: MariaDB
- 서버: `192.168.0.10`
- Port: `3306`
- Database: `lms`

### JDBC Driver

JDBC Driver는 Java 애플리케이션의 JDBC 호출을 각 DBMS가 이해할 수 있는 통신으로 연결한다.

```text
Spring / Java → JDBC → JDBC Driver → Database
```

| DBMS | Driver class |
| --- | --- |
| MariaDB | `org.mariadb.jdbc.Driver` |
| MySQL | `com.mysql.cj.jdbc.Driver` |
| PostgreSQL | `org.postgresql.Driver` |
| Oracle | `oracle.jdbc.OracleDriver` |

Spring Boot는 의존성과 JDBC URL을 바탕으로 Driver를 자동 판단할 수 있으므로 `driver-class-name`을 생략하기도 한다.

### Connection Pool과 HikariCP

DB 작업마다 Connection을 새로 생성하고 종료하면 네트워크 연결과 인증 비용이 반복된다. Connection Pool은 미리 만든 Connection을 빌려주고 사용 후 반환받아 재사용한다.

Spring Boot는 기본 Connection Pool 구현체로 일반적으로 HikariCP를 사용한다.

```yaml
spring:
  datasource:
    url: jdbc:mariadb://localhost:3306/lms
    username: root
    password: password
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
```

| 설정 | 역할 |
| --- | --- |
| `maximum-pool-size` | Pool이 관리할 수 있는 최대 Connection 수 |
| `minimum-idle` | 사용 중이지 않아도 유지할 최소 idle Connection 수 |
| `connection-timeout` | Connection을 얻기 위해 기다릴 최대 시간(ms) |

### Master/Slave DataSource

읽기 요청이 많은 서비스에서는 데이터베이스 역할을 분리하여 부하를 분산할 수 있다.

```text
Master DB: INSERT / UPDATE / DELETE
Slave DB:  SELECT
```

- Master: 데이터 변경을 담당하는 쓰기 중심 DB
- Slave: 복제된 데이터를 제공하는 읽기 중심 DB

서로 다른 DB 서버에 연결해야 하므로 접속 정보와 Connection Pool도 각각 구성한다.

```yaml
datasource:
  master:
    url: jdbc:mariadb://master-db:3306/lms
    username: user
    password: password
  slave:
    url: jdbc:mariadb://slave-db:3306/lms
    username: user
    password: password
```

### Replication

Master의 변경 사항을 Slave로 복제하는 과정을 Replication이라고 한다.

```text
INSERT / UPDATE / DELETE
          ↓
       Master
          │
     Replication
          ↓
        Slave
          ↑
        SELECT
```

## 원인

하나의 DB가 모든 읽기와 쓰기 요청을 처리하면 요청량이 증가할수록 Connection, CPU, 디스크 I/O 부하가 한곳에 집중된다. 특히 조회가 쓰기보다 많은 서비스에서는 읽기를 복제 DB로 분산하면 Master의 부담을 줄일 수 있다.

다만 Master의 변경 사항은 Slave에 즉시 반영된다고 보장할 수 없다. 복제 과정에는 시간이 필요하며 이 지연을 **Replication Lag**이라고 한다.

```text
1. Master에 사용자 A 저장
2. Slave로 아직 복제되지 않음
3. Slave에서 사용자 A 조회 → 결과 없음
4. Replication 완료
5. Slave에서 사용자 A 조회 가능
```

## 해결 방법

Spring에서는 Master와 Slave용 `DataSource`를 각각 Bean으로 등록하고, `AbstractRoutingDataSource`를 이용해 현재 트랜잭션의 성격에 맞는 DataSource를 선택하도록 구성할 수 있다.

대표적인 라우팅 기준은 트랜잭션의 `readOnly` 여부다.

```java
@Transactional
public void saveUser() {
    // Master 사용
}
```

```java
@Transactional(readOnly = true)
public User getUser() {
    // Routing 설정에 따라 Slave 사용
}
```

전체 흐름은 다음과 같다.

```text
application.yml
      ↓
Master / Slave DataSource 설정
      ↓
JDBC Driver
      ↓
HikariCP (Connection Pool)
      ↓
AbstractRoutingDataSource
   ↙                     ↘
Master                   Slave
INSERT/UPDATE/DELETE      SELECT
   │
   └──── Replication ────→
```

중요한 점은 `@Transactional(readOnly = true)` 자체가 Slave를 자동 선택하지 않는다는 것이다. `AbstractRoutingDataSource` 등의 별도 라우팅 로직이 있어야 `readOnly` 값을 기준으로 DataSource를 전환할 수 있다.

## 주의할 점

- 저장 직후 최신 데이터가 반드시 필요하면 Replication Lag을 고려해 Master에서 조회해야 한다.
- `@Transactional(readOnly = true)`는 읽기 전용 의도를 나타내지만 그 자체로 Slave 라우팅을 구성하지 않는다.
- Routing DataSource가 실제 Connection을 얻기 전에 라우팅 대상이 결정되어야 한다.
- 같은 트랜잭션 안에서 쓰기 후 읽기가 이어지면 읽기도 Master로 보내는 전략을 고려한다.
- Connection Pool 크기는 DB의 최대 Connection 수, 애플리케이션 인스턴스 수와 실제 부하를 함께 고려해 설정한다.
- Slave가 여러 대라면 라운드 로빈이나 가중치 방식 등의 읽기 분산 정책과 장애 대응이 추가로 필요하다.

### 면접·복습 포인트

- DataSource와 JDBC Driver는 각각 어떤 역할을 하는가?
- Connection Pool과 HikariCP를 사용하는 이유는 무엇인가?
- Master/Slave 구조가 읽기 부하를 줄이는 원리는 무엇인가?
- Replication Lag이 발생하면 어떤 문제가 생기는가?
- `AbstractRoutingDataSource`는 어떤 역할을 하는가?
- `@Transactional(readOnly = true)`만으로 Slave를 사용할 수 있는가?

## 한 줄 정리

> DataSource는 Spring이 DB Connection을 얻는 진입점이며, Master/Slave 구조에서는 쓰기와 읽기를 분리하고 Replication Lag을 고려한 라우팅 로직을 직접 구성해야 한다.

## 참고 자료

- [Spring Framework Javadoc - DataSourceTransactionManager](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/DataSourceTransactionManager.html)
- [Spring Framework Javadoc - AbstractRoutingDataSource](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/lookup/AbstractRoutingDataSource.html)
- [HikariCP Configuration](https://github.com/brettwooldridge/HikariCP#configuration-knobs-baby)
