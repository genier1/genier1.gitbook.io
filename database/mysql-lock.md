# Mysql Lock 정리

#### Shared Lock(S-lock)

* `SELECT ... FOR SHARE` 사용시 Shared Lock을 걸 수 있다.
* 레코드에 대해 읽기 잠금을 설정한다.
* 다른 세션에서 해당 레코드를 변경하지 못하게 한다.
* 다른 세션에서 Shared Lock이 걸린 레코드를 읽는 것은 가능하다.

#### Exclusive Lock

* `SELECT ... FOR UPDATE` 사용시 Exclusive Lock을 걸 수 있다.
* 레코드에 대해 쓰기 잠금을 설정한다.
  * `SELECT ... FOR UPDATE` 뿐 아니라 `SELECT ... FOR SHARE` 쿼리도 수행할 수 없다.
  * 위 두가지가 아닌 단순 SELECT 쿼리는 잠금에 걸리지 않는다.
* 예시

```java
BEGIN;
SELECT * FROM users WHERE id = 1 FOR SHARE;
```

해당 쿼리를 통해 users 테이블에 Shared Lock을 건다.

```java
SELECT * FROM users WHERE id = 1 FOR update;
```

Shared Lock이 걸린 레코드에 대해 Exclusive Lock을 걸게 되면 쿼리가 실행되지 않는다. 락 정보를 한번 확인해보자

```java
SELECT waiting_trx_id, waiting_lock_mode, blocking_trx_id, blocking_lock_mode, locked_table
FROM sys.innodb_lock_waits;

// 결과
waiting_trx_id, waiting_lock_mode, blocking_trx_id, blocking_lock_mode, locked_table
10141, X,REC_NOT_GAP, 281480239058048 S,REC_NOT_GAP, `users`
```

blocking\_lock\_mode는 S락, waiting\_lock\_mode가 X락으로 표시되고 있다.

`SELECT ... FOR SHARE` 을 통해 S락이 걸린 상태에서 X락을 걸려고 하다보니 S

REC\_NOT\_GAP락은 `Record lock without gap` 을 의미한다. 갭 락 없이 레코드 락만 획득했다는 의미이다.

#### Intention Lock

* InnoDB 스토리지 엔진에서 사용되는 테이블 수준의 락이다.
  * 반면, `Shared Lock(S Lock)`과 `Exclusive Lock(X Lock)`은 기본적으로 행(Row) 수준에서 적용되는 락이다.
* Intention Lock은 행 수준의 락과 테이블 수준의 락이 공존할 수 있도록 지원하는 다중 레벨 락킹(Multiple Granularity Locking) 메커니즘의 핵심 요소이다.
  * MGL은 row lock과 table lock이 공존할 수 있게 해주는데, 이를 통해 동시성을 향상시키면서도 데이터 무결성을 유지할 수 있다.
* Intention locking 프로토콜은 다음과 같다.
  * 트랜잭션이 `shared lock` 을 얻을 수 있으려면 우선 테이블에 대해서 `IS lock 혹은 더 강한 잠금` 을 가지고 있어야 한다.
  * 트랜잭션이 `exclusive lock` 을 얻을 수 있으려면 우선 테이블에 대해서 `IX 잠금` 을 가지고 있어야 한다.

#### Record Lock

* 레코드 자체만을 잠그는 역할
* InnoDB에서는 레코드 자체가 아니라 인덱스의 레코드를 잠근다.

레코드가 아닌 인덱스의 레코드를 잠그므로, 인덱스가 없는 칼럼에 대해 Lock을 걸때 매우 유의해야 한다.

인덱스가 없는 칼럼을 기준으로 레코드 락을 걸 경우 해당 쿼리를 처리하기 위해서는 테이블 전체를 스캔해야 한다. 따라서 InnoDB에서는 테이블 전체에 대해 테이블 락을 걸고 쿼리를 처리한다. 이로 인해 다른 세션에서 해당 테이블의 전혀 다른 레코드에 락을 걸 때도 해당 쿼리가 블로킹 된다.

```sql
select *
from users;

+--+---------------+
|id|name           |
+--+---------------+
|1 |John Doe       |
|2 |Jane Smith     |
|3 |Michael Johnson|
+--+---------------+
```

users 테이블에는 다음과 같은 정보가 있다.

```sql
START TRANSACTION;
SELECT * FROM users WHERE name = 'John Doe' FOR UPDATE;
```

name = 'John Doe'인 레코드에 대해 `SELECT FOR UPDATE` 를 통해 Exclusive Lock을 건 다음, 실제 users 테이블의 레코드에 어떻게 락이 걸리는지 확인해보자.

```sql
SELECT object_name, lock_type, lock_mode, lock_data, lock_status
FROM performance_schema.data_locks;

+-----------+---------+---------+----------------------+-----------+
|object_name|lock_type|lock_mode|lock_data             |lock_status|
+-----------+---------+---------+----------------------+-----------+
|users      |TABLE    |IX       |null                  |GRANTED    |
|users      |RECORD   |X        |supremum pseudo-record|GRANTED    |
|users      |RECORD   |X        |1                     |GRANTED    |
|users      |RECORD   |X        |2                     |GRANTED    |
|users      |RECORD   |X        |3                     |GRANTED    |
+-----------+---------+---------+----------------------+-----------+
```

name = 'John Doe'인 레코드에 락을 걸었으니 id=1인 레코드에만 락이 걸렸을 것이라고 생각했는데, lock\_data 칼럼을 확인해보면 users 테이블의 전체 데이터인 id =1,2,3에 모두 락이 걸린 것을 확인할 수 있다.

name 칼럼에 인덱스를 추가한 후 다시 한번 확인해보자.

```sql
CREATE INDEX idx_name ON users (name);

START TRANSACTION;
SELECT * FROM users WHERE name = 'John Doe' FOR UPDATE;

SELECT object_name, lock_type, lock_mode, lock_data, lock_status
FROM performance_schema.data_locks;
+-----------+---------+-------------+--------------------+-----------+
|object_name|lock_type|lock_mode    |lock_data           |lock_status|
+-----------+---------+-------------+--------------------+-----------+
|users      |TABLE    |IX           |null                |GRANTED    |
|users      |RECORD   |X            |'John Doe', 1       |GRANTED    |
|users      |RECORD   |X,REC_NOT_GAP|1                   |GRANTED    |
|users      |RECORD   |X,GAP        |'Michael Johnson', 3|GRANTED    |
+-----------+---------+-------------+--------------------+-----------+
```

인덱스를 추가하고 나니 원래 의도한대로 id=1, name = 'John Doe'인 레코드에만 X락이 걸린것을 확인할 수 있다.

#### Gap Lock

* 레코드 자체가 아니라 인접한 레코드 사이의 간격을 잠그는 락이다.
* 레코드와 레코드 사이에 새로운 레코드가 추가되는 것을 방지하는 용도이다.
* 갭락 그 자체보다는 넥스트 키 락의 일부로 자주 사용된다.
* 트랜잭션 격리 수준을 READ COMMITTED로 설정하거나, innodb\_locks\_unsafe\_for\_binlog를 활성화하면 검색 및 인덱스 스캔에 대한 갭락을 비활성화 할 수 있다.

#### Next-Key Lock

* 레코드 락과 갭 락을 합쳐놓은 형태의 잠금이다.
* 변경을 위해 검색하는 레코드에는 넥스트키락 방식으로 잠금이 걸린다.
* 갭 락이나 넥스트 키 락은 바이너리 로그에 기록되는 쿼리가 리플리카 서버에서 실행될 때 소스 서버에서 만들어낸 결과와 동일한 결과를 만들어내도록 보장해주는 것이 주목적이다.
* 그러나 넥스트 키 락과 갭락으로 인해 데드락이 발생하거나 다른 트랜잭션에서 대기가 자주 발생하므로 바이너리 로그 포맷을 Row 형태로 바꿔서 넥스트 키 락이나 갭락을 줄이는 것이 좋다.

```sql
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(50) NOT NULL
);

select *
from users;

+--+----------------+
|id|name            |
+--+----------------+
|29|Liam Evans      |
|30|Victoria Edwards|
|32|John Smith      |
+--+----------------+
```

위와 같은 users 테이블에 대해 next-key 락이 걸리는 케이스를 확인해보자.

```sql
START TRANSACTION;
SELECT * FROM users WHERE id >= 29 AND id <= 32 FOR UPDATE;
```

먼저 id를 기준으로 조회하는 쿼리를 수행한다.

```sql
START TRANSACTION;
INSERT INTO users (id, name) VALUES (31, 'John Smith');
```

첫번째 트랜잭션이 완료되지 않은 상태에서 id = 31인 레코드를 삽입하려고 하면 Lock이 발생한다. `performance_schema.data_locks` 테이블을 통해 Lock 정보를 확인해보자.

```sql
SELECT engine_transaction_id, object_name, index_name, lock_type, lock_mode, lock_data, lock_status
FROM performance_schema.data_locks;

+---------------------+-----------+----------+---------+----------------------+---------+-----------+
|engine_transaction_id|object_name|index_name|lock_type|lock_mode             |lock_data|lock_status|
+---------------------+-----------+----------+---------+----------------------+---------+-----------+
|10181                |users      |null      |TABLE    |IX                    |null     |GRANTED    |
|10181                |users      |PRIMARY   |RECORD   |X,GAP,INSERT_INTENTION|32       |WAITING    |
|10178                |users      |null      |TABLE    |IX                    |null     |GRANTED    |
|10178                |users      |PRIMARY   |RECORD   |X,REC_NOT_GAP         |30       |GRANTED    |
|10178                |users      |PRIMARY   |RECORD   |X                     |32       |GRANTED    |
+---------------------+-----------+----------+---------+----------------------+---------+-----------+
```

engine\_transaction\_id = 10181인 데이터를 보면 lock\_type = Record, lock\_mode = X,GAP,INSERT\_INTENTION 인 데이터가 있는 것을 확인할 수 있다.

락 타입이 레코드인 레코드락이 걸리면서 갭락이 걸려있는 것을 확인할 수 있다. 즉, Next-Key Lock이 걸린 것을 확인할 수 있다. 왜 Next-Key 락이 걸린걸까?

첫번째 트랜잭션에서 id가 30 이상, 32 이하인 row를 FOR UPDATE 쿼리를 통해 조회하면서 id가 30\~32인 레코드 사이의 갭에 대해 Exclusive락이 설정된 상황이다. 또한 id = 30, id = 32인 레코드에도 Exclusive락이 설정되었다.

이 상황에서 두번째 트랜잭션이 id가 30과 32의 사이인 31의 레코드를 삽입하려고 하면서 Next-Key Lock에 걸리게 된다.

#### **Insert Intention Lock**

* 행 삽입 전에 INSERT 연산에 의해 설정되는 일종의 갭 락이다.
* 같은 인덱스 갭에 삽입하려는 여러 트랜잭션이 서로 블로킹하지 않고 동시에 실행될 수 있도록 한다.
* Insert Intention Lock은 갭 락의 일종이지만, 다른 갭 락과는 호환된다. 즉, 여러 트랜잭션이 같은 갭에 대해 Insert Intention Lock을 동시에 가질 수 있다.
  * 이로 인해 삽입 작업 간의 동시성을 크게 향상된다.
* 하지만 Insert Intention Lock은 갭에 대한 Exclusive Lock이나 Shared Lock과 함께 쓰일수는 없다.

#### AUTO\_INCREMENT **Lock**

* AUTO\_INCREMENT 칼럼이 사용된 테이블에 동시에 여러 레코드가 INSERT 되는 경우, 저장되는 각 레코드는 중복되지 않고 저장된 순서대로 증가하는 일련번호 값을 가져야한다.
* InnoDB 엔진에서는 이를 위해 내부적으로 AUTO\_INCREMENT 락을 사용한다.
* AUTO\_INCREMENT 락을 명시적으로 걸고 획득하고 해제하는 방법은 없다.
* 자동증가 락은 아주짧은시간 동안 걸렸다가 해제되는 잠금이라서 대부분의 경우 문제가 되지 않는다.
* MySql 5.1 버전 이상에서는 innodb\_autoinc\_lock\_mode 변수를 통해 자동 증가 락의 작동 방식을 변경할 수 있다.
  * innodb\_autoinc\_lock\_mode=0 → 모든 Insert 쿼리는 자동증가 락을 사용한다.
  * innodb\_autoinc\_lock\_mode=1 → MySQL 서버가 Insert 되는 레코드의 건수를 정확히 예측할 수 있을때는 자동증가락을 사용하지 않고 훨씬 가볍고 빠른 래치를 이용해 처리한다.
  * innodb\_autoinc\_lock\_mode=2 → 절대 자동증가 락을 사용하지 않고 경량화된 래치를 사용한다. 그러나 STATEMENT 포맷의 바이너리 로그를 사용하는 복제에서는 소스 서버와 레플리카 서버의 자동증가 값이 달라질 수 있으므로 주의해야 한다.

#### 참고링크

* [https://jaeseongdev.github.io/development/2021/06/16/Lock의-종류-(Shared-Lock,-Exclusive-Lock,-Record-Lock,-Gap-Lock,-Next-key-Lock)/](https://jaeseongdev.github.io/development/2021/06/16/Lock%EC%9D%98-%EC%A2%85%EB%A5%98-\(Shared-Lock,-Exclusive-Lock,-Record-Lock,-Gap-Lock,-Next-key-Lock\)/)
* [https://velog.io/@soyeon207/DB-Lock-총정리-1-InnoDB-의-Lock](https://velog.io/@soyeon207/DB-Lock-%EC%B4%9D%EC%A0%95%EB%A6%AC-1-InnoDB-%EC%9D%98-Lock)
* [https://dev.mysql.com/doc/refman/8.3/en/innodb-locking.html](https://dev.mysql.com/doc/refman/8.3/en/innodb-locking.html)
* Real MySQL 8.0
* [https://medium.com/daangn/mysql-gap-lock-다시보기-7f47ea3f68bc](https://medium.com/daangn/mysql-gap-lock-%EB%8B%A4%EC%8B%9C%EB%B3%B4%EA%B8%B0-7f47ea3f68bc)
