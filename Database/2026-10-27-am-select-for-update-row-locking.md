# MariaDB SELECT FOR UPDATE와 행 잠금: 동시 신청 방지

- 학습일: 2026-10-27 (오전)
- 분야: Database / Spring Boot / MyBatis / MariaDB
- 선수 학습: 트랜잭션 격리 수준 READ COMMITTED vs REPEATABLE READ (2026-10-26)
- 학습 목표: 조회 후 수정하는 로직에서 발생하는 경쟁 상태를 이해하고, 트랜잭션 안에서 `SELECT ... FOR UPDATE`로 제어한다.

## 문제

교육과정의 **잔여 정원이 1명**일 때 사용자 A와 B가 동시에 수강 신청한다. 서버가 먼저 잔여 정원을 조회한 뒤 신청을 저장한다면, 두 사용자 모두 "1명 남음"을 읽어 정원을 초과할 수 있다.

**기대:** 잔여 정원이 1명이므로 한 사용자만 신청 성공.  
**실제 위험:** 조회와 수정이 별개로 실행되면 두 요청이 동일한 잔여 수를 읽고 각각 신청할 수 있다.

## 핵심 개념

### 1. 일반 SELECT와 SELECT ... FOR UPDATE

- 일반 `SELECT`: InnoDB에서 보통 MVCC 스냅샷을 읽으며, 다른 트랜잭션이 같은 행을 수정하지 못하도록 잠그지 않는다.
- `SELECT ... FOR UPDATE`: **잠금 읽기(locking read)**. 조회 대상에 배타적 잠금(X lock)을 확보하여 다른 트랜잭션의 충돌하는 잠금 읽기·수정을 기다리게 한다.
- 잠금은 일반적으로 **트랜잭션이 COMMIT/ROLLBACK될 때** 해제된다.
- `FOR UPDATE`는 스냅샷을 재사용하는 일반 SELECT와 달리 **현재 읽기(current read)** 성격을 갖는다.

### 2. 두 요청의 실행 순서

초기 잔여 정원 `remaining_seats = 1`, 같은 행(`course_id = 1`)을 대상으로 한다.

```text
시간    트랜잭션 A                         트랜잭션 B
T1      BEGIN                             BEGIN
T2      SELECT ... FOR UPDATE → 1
T3                                        SELECT ... FOR UPDATE → 대기
T4      UPDATE remaining_seats = 0
T5      COMMIT → 잠금 해제
T6                                        대기 종료, 현재 값 0 조회
T7                                        정원 없음 → ROLLBACK
```

따라서 트랜잭션 A만 신청을 완료한다. B는 A의 잠금이 풀릴 때까지 대기한 뒤 잔여 정원을 다시 검사한다.

**전제:** InnoDB 테이블, 동일한 행을 인덱스로 찾는 조건, 명시적 트랜잭션, 성공 시 신청 저장과 잔여 정원 변경이 같은 트랜잭션에 포함된 경우다.

## 원인

다음과 같은 코드는 **조회 후 수정(check-then-act)** 경쟁 상태를 만들 수 있다.

```java
// 잘못된 예: 두 요청이 동시에 remaining=1을 읽을 수 있다.
int remaining = mapper.selectRemainingSeats(courseId);
if (remaining > 0) {
    mapper.decreaseSeat(courseId);
    mapper.insertEnrollment(courseId, userId);
}
```

일반 SELECT만으로는 다른 요청의 수정이 차단되지 않는다. 어제 배운 `REPEATABLE READ`도 일반 SELECT의 스냅샷 일관성을 제공하는 것이지, 이 업무 로직의 동시 신청을 자동으로 직렬화하지 않는다.

## 해결 방법

### 1. 실습용 테이블 준비

```sql
CREATE TABLE cs_course_capacity (
    course_id BIGINT PRIMARY KEY,
    remaining_seats INT NOT NULL
) ENGINE=InnoDB;

INSERT INTO cs_course_capacity(course_id, remaining_seats)
VALUES (1, 1);
```

### 2. 두 DB 세션으로 재현

**세션 A**

```sql
START TRANSACTION;

SELECT remaining_seats
FROM cs_course_capacity
WHERE course_id = 1
FOR UPDATE; -- 1, 해당 행 잠금 확보

UPDATE cs_course_capacity
SET remaining_seats = remaining_seats - 1
WHERE course_id = 1;

-- 여기서 세션 B의 SELECT를 실행해 대기 상태를 관찰
COMMIT;
```

**세션 B — A의 COMMIT 이전에 시작**

```sql
START TRANSACTION;

SELECT remaining_seats
FROM cs_course_capacity
WHERE course_id = 1
FOR UPDATE; -- A의 COMMIT까지 대기, 이후 0 조회

-- 잔여 정원이 0이므로 UPDATE하지 않는다.
ROLLBACK;
```

**실습 순서:** A의 `UPDATE`까지 실행 → B의 `SELECT ... FOR UPDATE` 실행(대기 확인) → A의 `COMMIT` 실행 → B가 0을 읽는지 확인한다. 두 세션은 서로 다른 DB 연결이어야 한다.

### 3. Spring Boot + MyBatis 실무 코드

```java
@Mapper
public interface CourseCapacityMapper {

    @Select("""
        SELECT remaining_seats
        FROM cs_course_capacity
        WHERE course_id = #{courseId}
        FOR UPDATE
        """)
    Integer selectRemainingForUpdate(@Param("courseId") Long courseId);

    @Update("""
        UPDATE cs_course_capacity
        SET remaining_seats = remaining_seats - 1
        WHERE course_id = #{courseId}
          AND remaining_seats > 0
        """)
    int decreaseSeat(@Param("courseId") Long courseId);

    // 실제 프로젝트에서는 신청 테이블의 유니크 키 등도 함께 검증
    @Insert("""
        INSERT INTO cs_enrollment(course_id, user_id)
        VALUES (#{courseId}, #{userId})
        """)
    int insertEnrollment(@Param("courseId") Long courseId,
                         @Param("userId") Long userId);
}
```

```java
@Service
@RequiredArgsConstructor
public class EnrollmentService {

    private final CourseCapacityMapper mapper;

    @Transactional
    public void enroll(Long courseId, Long userId) {
        Integer remaining = mapper.selectRemainingForUpdate(courseId);

        if (remaining == null) {
            throw new IllegalArgumentException("교육과정이 없습니다.");
        }
        if (remaining <= 0) {
            throw new IllegalStateException("신청 정원이 마감되었습니다.");
        }

        if (mapper.decreaseSeat(courseId) != 1) {
            throw new IllegalStateException("잔여 정원 변경에 실패했습니다.");
        }

        mapper.insertEnrollment(courseId, userId);
        // 정상 종료 시 COMMIT; RuntimeException 발생 시 기본적으로 ROLLBACK
    }
}
```

- `cs_enrollment`는 기존 신청 테이블이 있다는 가정의 예시이며, 실제 실습 시에는 해당 테이블이 필요하다.
- `@Transactional` 메서드는 **다른 Spring Bean에서 프록시를 거쳐 호출**해야 한다.
- 같은 트랜잭션에서 정원 감소와 신청 INSERT를 처리하므로 INSERT 실패 시 정원 감소도 롤백할 수 있다.

### 4. 대안: 단일 조건부 UPDATE

잔여 정원 감소만 원자적으로 처리하면 충분한 경우에는 아래 SQL이 더 단순할 수 있다.

```sql
UPDATE cs_course_capacity
SET remaining_seats = remaining_seats - 1
WHERE course_id = 1
  AND remaining_seats > 0;
```

영향받은 행 수가 **1이면 성공, 0이면 정원 없음**으로 판단한다. 이 방식도 INSERT와 같은 트랜잭션으로 묶어야 한다. 별도의 선행 `SELECT FOR UPDATE`가 항상 필요한 것은 아니다.

## 주의할 점

- `SELECT ... FOR UPDATE`는 **트랜잭션 안에서** 사용해야 한다. 자동 커밋 상태에서 단독으로 실행하면 잠금이 다음 SQL까지 유지되지 않는다.
- `WHERE` 조건에 적절한 인덱스가 없으면 예상보다 많은 레코드 또는 인덱스 범위를 잠글 수 있다. `REPEATABLE READ`에서는 갭/넥스트키 잠금도 고려한다.
- 잠금 대기가 길어지면 타임아웃이 발생할 수 있고, 여러 행을 서로 다른 순서로 잠그면 **데드락**이 발생할 수 있다.
- 잠금 범위 안에 느린 외부 API 호출을 넣지 말고 트랜잭션을 짧게 유지한다.
- 중복 신청 방지는 정원 잠금과 별개다. 신청 테이블에 `UNIQUE(course_id, user_id)` 같은 제약을 둔다.
- 위 설명은 **MariaDB/InnoDB** 기준이며 다른 DBMS의 잠금 동작은 다를 수 있다.

### 확인 문제

**Q.** 정원이 1명인 강좌에 A와 B가 동시에 신청한다. 두 요청 모두 `@Transactional` 안에서 `SELECT ... FOR UPDATE`로 같은 `course_id`의 잔여 정원을 읽는다. A가 먼저 잠금을 확보해 잔여 정원을 0으로 변경하고 커밋했다. B는 무엇을 읽으며, 신청해야 할까?

**정답 및 해설:** B는 A가 잠금을 해제할 때까지 대기하고, 이후 현재 값 **0**을 읽는다. 정원이 없으므로 INSERT하지 않고 신청을 거절해야 한다. 단순한 일반 SELECT였다면 두 요청 모두 이전 값 1을 읽는 경쟁 상태가 생길 수 있다.

**짧은 실습:** 위 세션 A/B SQL을 순서대로 실행해 B의 대기와 커밋 후 조회값을 확인한다. 운영 DB가 아닌 학습용 DB에서 실행한다.

## 한 줄 정리

> `SELECT ... FOR UPDATE`는 트랜잭션 안에서 조회한 행에 잠금을 확보해 동시 수정 경쟁을 제어하지만, 정원 차감처럼 단순한 작업은 조건부 UPDATE도 유효한 대안이다.

## 참고 자료

- [MariaDB — FOR UPDATE](https://mariadb.com/docs/server/reference/sql-statements/data-manipulation/selecting-data/for-update)
- [MariaDB — InnoDB Lock Modes](https://mariadb.com/kb/en/innodb-lock-modes/)
- [Spring Framework — Declarative Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative.html)
