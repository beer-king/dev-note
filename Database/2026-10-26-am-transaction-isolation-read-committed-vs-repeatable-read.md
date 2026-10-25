# DB 트랜잭션 격리 수준: READ COMMITTED vs REPEATABLE READ

- 학습일: 2026-10-26 (오전)
- 분야: Database / Spring Boot / MyBatis / MariaDB
- 선수 학습: Spring `@Transactional` 프록시와 트랜잭션 경계 (2026-10-25)
- 학습 목표: 같은 트랜잭션에서 SELECT를 두 번 실행했을 때 값이 달라지는 이유를 설명한다.

## 문제

교육 신청 건수를 확인하는 Spring Boot API에서 동일한 행을 두 번 조회했는데, 그 사이 다른 사용자가 데이터를 수정했다. 첫 조회와 두 번째 조회의 결과는 항상 같을까?

**기대:** 같은 트랜잭션 안이라면 조회 결과가 항상 동일하다.  
**실제:** DB의 **트랜잭션 격리 수준(Isolation Level)**, 조회 방식, 동시 변경 여부에 따라 달라진다.

## 핵심 개념

### 1. 트랜잭션 격리 수준이란?

여러 트랜잭션이 동시에 실행될 때 **다른 트랜잭션이 변경한 데이터를 언제 볼 수 있는지**를 결정하는 규칙이다. ACID의 I(Isolation)에 해당한다.

이번에는 다음 두 수준만 비교한다.

| 격리 수준 | 일반 SELECT에서 다른 트랜잭션의 커밋을 보는 시점 | Non-repeatable read |
| --- | --- | --- |
| `READ COMMITTED` | **각 SELECT 문**이 시작할 때 기준으로 커밋된 데이터 | 발생 가능 |
| `REPEATABLE READ` | **같은 트랜잭션의 첫 일관성 읽기**에서 만들어진 스냅샷을 재사용(InnoDB 기준) | 일반적인 일관성 SELECT에서는 방지 |

- **Non-repeatable read(반복 불가능 읽기):** 트랜잭션 A가 같은 행을 두 번 읽는 사이 B가 수정·커밋하여 A의 두 조회 결과가 달라지는 현상.
- **MVCC(Multi-Version Concurrency Control):** DB가 여러 버전의 행을 관리하여 일반 SELECT가 적절한 시점의 데이터 버전을 읽게 하는 방식.
- MariaDB의 일반적인 **InnoDB 기본 격리 수준은 `REPEATABLE READ`**다. 단, 서버/세션 설정을 확인해야 한다.

### 2. 동작 원리

초기 잔액 `100`인 행을 가정하자.

```text
시간      트랜잭션 A                         트랜잭션 B
T1        BEGIN
T2        SELECT amount → 100
T3                                             BEGIN
T4                                             UPDATE amount = 200
T5                                             COMMIT
T6        SELECT amount → ?
T7        COMMIT
```

InnoDB의 **일반 SELECT(잠금 없는 일관성 읽기)** 기준:

- A가 `READ COMMITTED`이면 T6에 **200**을 볼 수 있다.
- A가 `REPEATABLE READ`이면 T6에도 **100**을 본다. A가 종료된 뒤 새 트랜잭션에서 읽으면 200을 볼 수 있다.

이는 SQL 실행이 실패한 것이 아니라 **격리 수준에 따른 정상 동작**이다.

## 원인

`READ COMMITTED`에서는 각 일반 SELECT가 새로 커밋된 변경을 볼 수 있어 같은 행의 값이 달라질 수 있다. 반면 InnoDB `REPEATABLE READ`의 일반 SELECT는 트랜잭션 내 동일한 읽기 스냅샷을 사용한다.

단, **모든 SELECT가 스냅샷만 읽는 것은 아니다.** `SELECT ... FOR UPDATE` 같은 잠금 읽기나 `UPDATE`는 현재 데이터와 잠금 규칙에 따라 동작하므로 일반 SELECT와 구분해야 한다.

## 해결 방법

### 1. MariaDB 두 세션으로 재현하기

학습용 DB에서만 아래 테이블을 준비한다.

```sql
CREATE TABLE cs_account (
    id BIGINT PRIMARY KEY,
    amount INT NOT NULL
) ENGINE=InnoDB;

INSERT INTO cs_account (id, amount) VALUES (1, 100);
```

**세션 A — READ COMMITTED**

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;

SELECT amount FROM cs_account WHERE id = 1; -- 100
-- 여기서 세션 B를 실행한 다음 다시 조회
SELECT amount FROM cs_account WHERE id = 1; -- 200
COMMIT;
```

**세션 B — A의 두 SELECT 사이에 실행**

```sql
START TRANSACTION;
UPDATE cs_account SET amount = 200 WHERE id = 1;
COMMIT;
```

다음 실습에서는 값을 100으로 되돌린 뒤, **세션 A의 격리 수준만 `REPEATABLE READ`로 변경**하고 같은 순서로 실행한다. 이때 A의 두 번째 일반 SELECT는 **100**이다.

```sql
-- 이전 트랜잭션을 끝낸 후 초기화
UPDATE cs_account SET amount = 100 WHERE id = 1;

-- 세션 A
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT amount FROM cs_account WHERE id = 1; -- 100
-- 세션 B가 UPDATE amount=200 후 COMMIT
SELECT amount FROM cs_account WHERE id = 1; -- 100
COMMIT;
```

**주의:** SQL을 한 세션에서 위에서 아래로 한 번에 실행하면 동시 실행 상황이 재현되지 않는다. 세션 A의 첫 SELECT와 둘째 SELECT 사이에 세션 B의 COMMIT을 완료해야 한다.

### 2. Spring Boot + MyBatis에서 지정하기

```java
@Service
@RequiredArgsConstructor
public class AccountQueryService {
    private final AccountMapper accountMapper;

    @Transactional(
        readOnly = true,
        isolation = Isolation.REPEATABLE_READ
    )
    public int[] readTwice(Long id) {
        int first = accountMapper.selectAmount(id);
        // 이 사이에 다른 요청이 같은 행을 UPDATE/COMMIT할 수 있음
        int second = accountMapper.selectAmount(id);
        return new int[] { first, second };
    }
}
```

```java
@Mapper
public interface AccountMapper {
    @Select("SELECT amount FROM cs_account WHERE id = #{id}")
    int selectAmount(Long id);
}
```

Spring의 `Isolation`은 `org.springframework.transaction.annotation.Isolation`이다. 다른 Spring Bean에서 이 Service를 호출해 트랜잭션 프록시가 적용되고, 사용하는 트랜잭션 매니저와 DB가 해당 격리 수준을 지원해야 한다.

### 3. 실무에서 선택하는 기준

- 일반적인 조회 API에서는 무조건 격리 수준을 올리기보다 **DB 기본값과 데이터 정합성 요구사항**을 먼저 확인한다.
- 한 작업에서 같은 행을 반복 조회하며 일관된 결과가 필요하다면 `REPEATABLE READ`의 스냅샷 특성을 이해하고 사용한다.
- **동시 수정으로 인한 덮어쓰기(Lost Update)**를 방지해야 한다면 격리 수준만 믿지 말고 조건부 `UPDATE`, 낙관적 락(버전 컬럼), 필요 시 잠금 읽기 등 별도 동시성 제어를 검토한다.

## 주의할 점

- 위 설명은 **MariaDB/InnoDB의 일반 SELECT** 기준이다. DBMS별 격리 수준 구현이 완전히 동일하지 않다.
- `@Transactional(isolation = ...)` 설정은 **새로 시작하는 트랜잭션**에 적용된다. 기본 `REQUIRED`로 기존 트랜잭션에 참여하면 바깥 트랜잭션의 격리 수준을 따르는 것이 일반적이다.
- `readOnly = true`는 읽기 전용 의도 및 최적화 힌트이며, 모든 환경에서 쓰기를 물리적으로 차단한다고 가정하면 안 된다.
- `REPEATABLE READ`가 모든 동시성 문제를 해결하는 것은 아니다. 잠금 읽기, 범위 조회, 업데이트 시에는 별도 고려가 필요하다.
- 실습용 테이블은 테스트 DB에서만 생성한다.

### 확인 문제

**Q.** MariaDB/InnoDB에서 A가 `READ COMMITTED` 트랜잭션을 시작해 일반 SELECT로 `amount=100`을 읽었다. B가 같은 행을 `200`으로 변경하고 커밋했다. A가 같은 SELECT를 다시 실행하면 얼마일까? A를 `REPEATABLE READ`로 바꾸면 결과는?

**정답 및 해설:** `READ COMMITTED`는 **200**, `REPEATABLE READ`는 **100**(첫 일관성 읽기에서 100을 본 경우). 전자는 SELECT마다 새 커밋 상태를 볼 수 있고, 후자는 트랜잭션의 일관성 읽기 스냅샷을 재사용하기 때문이다.

**짧은 실습:** 위 SQL을 DB 클라이언트의 서로 다른 연결 두 개로 실행해 두 결과를 직접 확인한다.

## 한 줄 정리

> InnoDB의 일반 SELECT는 READ COMMITTED에서 문장별 커밋 상태를, REPEATABLE READ에서 트랜잭션의 일관된 스냅샷을 읽는다.

## 참고 자료

- [MariaDB — SET TRANSACTION](https://mariadb.com/kb/en/set-transaction/)
- [MariaDB — InnoDB Isolation Levels](https://mariadb.com/kb/en/innodb-isolation-levels/)
- [Spring Framework — @Transactional](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
