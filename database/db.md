# DB 커넥션 풀

Spring Boot 2.0 이후 부터는 기본 커넥션 풀을 HikariCP로 사용한다. HikariCP를 사용할 수 없는 경우 Tomcat DBCP → Apachde Commons DBCP → Oracle UCP 순으로 선택한다.

HikarCP란 DBC의 커넥션 풀 오픈소스 라이브러리인 JDBC DataSource의 구현체이다. 다른 커넥션 풀 프레임워크에 비해 압도적인 성능을 보여준다. 이후 설명하는 부분은 모두 Hikari CP 기준이다.

{% embed url="https://github.com/genier1/genier1.gitbook.io/assets/19471818/84a4a310-5a1d-4fb4-8e53-3e4b2c6fea4e" %}
출처: [https://github.com/brettwooldridge/HikariCP](https://github.com/brettwooldridge/HikariCP)
{% endembed %}

#### HikariCP 기본 설정

```sql
spring:
  datasource:
    type: com.zaxxer.hikari.HikariDataSource
    hikari:
      minimumIdle: 10
      maximumPoolSize: 10
      idleTimeout: 600000
      maxLifetime: 1800000
      connectionTimeout: 30000
      poolName: MyHikariCP
      autoCommit: true
      readOnly: false
      connectionTestQuery: SELECT 1
      validationTimeout: 5000
```

* **minimum-idle**: Connection Pool에 유지 가능한 최소 커넥션 개수
* **maximum-pool-size**: Connection Pool에 유지 가능한 최대 커넥션 개수
* **idle-timeout**: Connection이 Poll에서 유휴상태(사용하지 않는 상태)로 남을 수 있는 최대 시간(초)
* **pool-name**: Connction Pool 이름
* **max-lifetime**: Connection의 최대 유지 가능 시간(초)
* **connection-timeout**: Pool에서 Connection을 구할 때 대기시간(초), 대기시간안에 구하지 못하면 Exception 발생

#### 적정 커넥션 풀 사이즈

커넥션 풀을 무작정 늘리는 건 좋지 않다. CPU 코어가 하나뿐인 컴퓨터도 수십 또는 수백 개의 스레드를 '동시에' 지원할 수 있는것 처럼 보이지만 사실 눈속임이다. 하나의 코어는 한 번에 하나의 스레드만 실행할 수 있다.

스레드 수가 CPU 코어 수를 초과하면 컨텍스트 스위칭, 스케줄링 오버헤드로 인해 오히려 속도가 느려지게 된다.

Hikari CP에서 권장하는 커넥션 풀 수는 다음과 같다.

* **connections = ((core\_count \* 2) + effective\_spindle\_count)**
  * core\_count는 CPU 코어 개수이다.
  * effective\_spindle\_count는 유효한 디스크 수 정도로 번역할 수 있다.
    * 하드디스크가 2개 있다고 가정하면 effective\_spindle\_count는 2라고 생각할 수 있지만, 사실은 캐시 적중율에 따라 달라진다.
    * 예를 들어, 캐시 적중율이 50%여서 DB에서 하드디스크 하나에만 연결을 수행하고 있다면 effective\_spindle\_count = 1이다.
    * 캐시 적중율이 100%여서 DB에서 어떤 하드디스크에도 연결을 수행하고 있지 않다면 effective\_spindle\_count = 0이다.
  * 예를 들어, 4코어 i7 서버와 한개의 하드디스크가 있다면, 적절한 커넥션 풀 개수는 10개다. ((4 \* 2) + 1)
  * 커넥션 풀 개수가 10개면 얼핏 봤을 때 부족해 보일 수 있으나, 3000명의 프론트엔드 사용자가 간단한 쿼리를 6000 TPS로 실행하는 것을 감당 가능한 수준이며 커넥션 풀 개수를 늘리면 오히려 TPS가 감소한다.

[Hikari CP의 글](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing#axiom-you-want-a-small-pool-saturated-with-threads-waiting-for-connections)에 따르면 아무리 큰 서비스라고 하더라도 스레드 수가 100개는 과도하다고 한다. Hikari CP에서는 최대 수십 개의 커넥션으로 구성된 작은 풀을 설정하는 것을 권장한다. 만약 커넥션 풀이 모두 소비되었다면, 애플리케이션 쓰레드들이 풀로부터 커넥션을 받기 위해 블록되도록 하는 것을 권장한다.

#### 데드락 방지를 위한 최소 커넥션 풀 개수

커넥션풀을 사용할 때, 각 스레드가 서로의 DB Connection이 반납되기만을 무한정 대기하는 DeadLock 상황을 주의해야 한다. 이 부분을 방지하기 위해서 Hikari CP의 문서에서 권장하는 최소 커넥션 풀

데드락을 방지하기 위한 최소 커넥션 풀 개수는 다음과 같다.

* **pool size = Tn x (Cm - 1) + 1**
* Tn : 전체 Thread 갯수
* Cm : 하나의 Task에서 동시에 필요한 Connection 수

예를 들어, 각각 어떤 작업을 수행하기 위해 4개의 연결이 필요한 (_Cm=4_) 3개의 스레드 (_Tn=3_)가 있다고 해보자. 그렇다면 이 애플리케이션에 연결되어야 하는 최소 커넥션 수는 10개다. (3 x (4 - 1) + 1)

추가적으로 배민 블로그 글에서 제시한 Deadlock 발생 가능성 감지 방법이다.

* **HikariCP의 Maximum Pool Size을 1로 설정한 다음 1건씩 Query를 실행해본다.**
  * 만약 정상적으로 실행되지 않고, connection timeout과 같은 에러가 발생한다면 Dead lock 발생 가능성이 있는 코드이다.
* **Nested Transaction을 사용하지 않는다.** 보이지 않는 dead lock을 유발할 수 있다.

#### 참고링크

* [https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
* [https://techblog.woowahan.com/2663/](https://techblog.woowahan.com/2663/)
