# DB 트랜잭션 격리수준

트랜잭션 격리수준(isolation level)이란 동시에 여러 트랜잭션이 처리될 때, 트랜잭션끼리 얼마나 서로 고립되어 있는지를 나타내는 것이다.

즉, 격리수준을 통해 특정 트랜잭션이 다른 트랜잭션에 변경한 데이터를 어느 수준까지 볼 수 있도록 허용할지를 결정한다.

트랜잭션 격리 수준이 높을수록 동시에 실행되는 트랜잭션들이 서로에게 영향을 미치지 않도록 격리할 수 있으나, 그 과정에서 다른 데이터 잠금이 해제될 때까지 기다려야 할 가능성이 높아지므로 성능이 저하될 수 있다.

격리수준은 총 4가지가 있다.

* READ UNCOMMITTED(커밋되지 않은 읽기)
* READ COMMITTED(커밋된 읽기)
* REPEATABLE READ(반복 가능한 읽기)
* SERIALIZABLE(직렬화 가능)

트랜잭션 격리수준에 따라 아래와 같은 현상이 발생할 수 있다. 해당 부분은 각 격리수준을 설명할 때 함께 설명한다.

![트랜잭션 격리수준.webp](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8a99740f-8e04-4c81-8499-1d73542b0fec/%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%AB%E1%84%8C%E1%85%A2%E1%86%A8%E1%84%89%E1%85%A7%E1%86%AB\_%E1%84%80%E1%85%A7%E1%86%A8%E1%84%85%E1%85%B5%E1%84%89%E1%85%AE%E1%84%8C%E1%85%AE%E1%86%AB.webp)

#### 현재 격리수준 조회 방법

현재 DB의 격리수준을 조회하기 위해서는 아래의 쿼리로 확인할 수 있다.

```sql
mysql> show variables like 'transaction_isolation';
+-----------------------+-----------------+
| Variable_name         | Value           |
+-----------------------+-----------------+
| transaction_isolation | REPEATABLE-READ |
+-----------------------+-----------------+
```

현재 DB의 격리수준이 REPEATABLE-READ임을 알 수 있다.

테스트를 위해 `tb_product`라는 테이블 생성한다.

```sql
CREATE TABLE tb_product
(
    id   int(10) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name varchar(32) NOT NULL,
    price int(10) NOT NULL
);
```

#### Read Uncommited

```sql
mysql> SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

mysql> show variables like 'transaction_isolation';
+-----------------------+------------------+
| Variable_name         | Value            |
+-----------------------+------------------+
| transaction_isolation | READ-UNCOMMITTED |
+-----------------------+------------------+
```

우선 위와 같이 해당 세션의 격리수준을 READ UNCOMMITTED로 설정한다. 그리고 나서 격리수준을 확인하면 해당 세션의 격리수준이 READ-UNCOMMITTED로 설정된 것을 확인할 수 있다.

이제 아래와 같이 트랜잭션을 시작하고 데이터를 추가한후 롤백해본다.

```sql
# 1번 세션
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO tb_product (id, name, price) VALUES (1, '농구공', 20000);
Query OK, 1 row affected (0.00 sec)

mysql> rollback;
Query OK, 0 rows affected (0.00 sec)
```

한편, 2번째 세션을 열어 똑같이 세션의 격리수준을 READ UNCOMMITTED로 설정한 후, 트랜잭션을 시작하고 조회쿼리를 입력한다.

```sql
mysql> start transaction;
Query OK, 0 rows affected (0.01 sec)

# 1번 세션에서 데이터 삽입 전 데이터 조회
mysql> select * from tb_product;
Empty set (0.01 sec)

# 1번 세션에서 데이터 삽입 후 데이터 조회
mysql> select * from tb_product;
+----+-----------+-------+
| id | name      | price |
+----+-----------+-------+
|  1 | 농구공      | 20000 |
+----+-----------+-------+
1 row in set (0.00 sec)

# 1번 세션에서 롤백 후 데이터 조회
mysql> select * from tb_product;
Empty set (0.00 sec)

mysql> commit;
```

transaction을 시작하고 나서 같은 조회쿼리를 연속으로 입력했는데 갑자기 Row가 추가되었다가 사라지는 것을 볼 수 있다.

위와 같이 다른 트랜잭션에서 처리한 작업이 커밋되지 않았음에도 불구하고 다른 트랜잭션에서 볼 수 있게 되는 현상을 Dirty Read라고 한다. 격리수준이 READ UNCOMMITED 상태일 때는 Dirty Read가 발생하는 것을 확인할 수 있다.

#### Read Commited

격리수준이 Read Commited 일때는 Read Uncommited처럼 Dirty Read는 발생하지 않는다. 그러나 다른 상황이 발생한다. 이번에는 데이터를 추가한 후 커밋을 해보자.

```sql
# 1번 세션
mysql> SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO tb_product (id, name, price) VALUES (1, '농구공', 20000);
Query OK, 1 row affected (0.00 sec)

mysql> commit;
Query OK, 0 rows affected (0.00 sec)
```

다른 세션을 열어 똑같이 세션의 격리수준을 READ COMMITTED로 설정한 후, 트랜잭션을 시작하고 조회쿼리를 입력한다.

```sql
# 2번 세션 
mysql> SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
mysql> start transaction;
Query OK, 0 rows affected (0.01 sec)

# 1번 세션에서 커밋하기 전에 데이터 조회
mysql> select * from tb_product;
Empty set (0.01 sec)

# 1번 세션에서 커밋한 후 데이터 조회
mysql> select * from tb_product;
+----+-----------+-------+
| id | name      | price |
+----+-----------+-------+
|  1 | 농구공      | 20000 |
+----+-----------+-------+
1 row in set (0.00 sec)

mysql> commit;
```

동일 트랜잭션 내에서 커밋되지 않은 데이터는 읽을수 없지만 다른 트랜잭션에서 커밋한 데이터가 동일 트랜잭션 내에서 보이는 것을 확인할 수 있다.

하나의 트랜잭션내에서 동일한 SELECT 쿼리를 실행했을 때 항상 같은 결과를 보장해야 한다는 "REPEATABLE READ" 정합성에 어긋나게 된다.

#### Repetable Read

REPEATABLE READ는 MySQL의 InnoDB 스토리지 엔진에서 기본적으로 사용되는 격리 수준이다. Repetable Read에서는 NON-REPEATABLE READ 이슈가 발생하지 않는다.

그러나 Repetable Read 격리 수준에서는 Phantom Read라는 현상이 발생할 수 있다. 단, Mysql InnoDB에서는 Phantom Read가 발생하지 않는데, 해당 이슈는 바로 아래에서 설명한다.

Phantom Read란 Select for update와 같이 쓰기 잠금을 거는 경우 다른 트랜잭션에서 수행한 변경 작업에 의해 레코드가 보였다가 안 보였다가 하는 현상을 말한다. InnoDB에서는 격리수준이 Repetable Read이더라도 \*\*MVCC(Multi Version Concurrency Control)\*\*를 이용해 트랜잭션 내에서 첫 조회시 만들어진 SNAPSHOT을 기반으로 같은 조회 결과를 반환한다. 따라서 Phantom Read가 발생하지 않는다.

```sql
# 테스트를 위한 데이터 삽입
mysql> SET SESSION TRANSACTION ISOLATION LEVEL REPETABLE READ;
mysql> INSERT INTO tb_product (id, name, price) VALUES (1, '농구공', 20000);
mysql> select * from tb_product where id = 1;
+----+-----------+-------+
| id | name      | price |
+----+-----------+-------+
|  1 | 농구공     | 20000 |
+----+-----------+-------+
```

위의 데이터를 통해 Mysql InnoDB에서 Phantom Read가 발생하는지 알아보자.

우선, 1번 세션에서 Transaction을 시작하고 쓰기 잠금을 건다.

```sql
# 1번 세션에서 먼저 트랜잭션 시작
mysql> start transaction;
mysql> select * from tb_product where id = 1 for update;
+----+-----------+-------+
| id | name      | price |
+----+-----------+-------+
|  1 | 농구공      | 20000 |
+----+-----------+-------+
```

그 다음, 2번 세션에서 트랜잭션을 시작하고 농구공의 price를 수정한 다음 커밋해보자.

```sql
# 1번 세션에서 먼저 트랜잭션을 시작한 후 2번 세션에서 트랜잭션을 시작하고 
# 농구공의 가격을 업데이트 한다.
mysql> start transaction;
mysql> update tb_product set price = 30000 where id = 1;
mysql> select * from tb_product where id = 1;
+----+-----------+-------+
| id | name      | price |
+----+-----------+-------+
|  1 | 농구공      | 30000 |
+----+-----------+-------+
mysql> commit;
```

다시, 1번 세션으로 돌아가 쓰기 잠금을 걸어보자.

```sql
# 2번 세션에서 농구공의 가격을 업데이트 한 후
# 트랜잭션이 유지되고 있는 1번 세션으로 돌아와 농구공의 가격을 조회한다.
mysql> select * from tb_product where id = 1 for update;
+----+-----------+-------+
| id | name      | price |
+----+-----------+-------+
|  1 | 농구공      | 20000 |
+----+-----------+-------+
```

Phantom Read가 발생했다면 농구공의 price가 30000으로 조회되어야 했지만, InnoDB의 MVCC 덕분에 Phantom Read가 발생하지 않은 것을 확인할 수 있다.

#### Serializable

Serializable는 위에서 언급했던 Dirty Read, Repetable Read, Phantom Read가 모두 발생하지 않는다. 그러나 Serializable은 읽기 시에도 테이블 락을 걸어버린다는 치명적인 단점이 있다.

따라서 실무에서는 왠만해서는 Serializable을 격리수준으로 사용하지 않는다.
