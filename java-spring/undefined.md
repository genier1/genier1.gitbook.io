# 톰캣 스레드풀

Spring Boot 3.2 기준

#### 톰캣 기본 설정

```sql
server:
  port: 8080
  servlet:
    context-path: /
  tomcat:
    threads:
      max: 200
    max-connections: 8192
    accept-count: 100
    http1:
      enabled: true
    accesslog:
      enabled: false
    ssl:
      enabled: false
```

* `server.tomcat.threads.max`
  * 톰캣의 최대 스레드 수, 기본값은 200
* `server.tomcat.max-connections`
  * 톰캣의 톰캣의 최대 연결 수, 기본값은 8192
* `server.tomcat.http1.enabled`
  * HTTP1.1 커넥터의 활성화 여부를 설정. 기본값은 true다.
* `server.tomcat.accept-count`
  * TCP 백로그 큐의 최대 크기를 설정. 기본값은 100
  * 이 값을 초과하는 연결 요청이 들어오면, 클라이언트는 연결 거부 오류를 받는다.
* `server.tomcat.accesslog.enabled`: 액세스 로그의 활성화 여부를 설정한다. 기본값은 false
* `server.tomcat.ssl.enabled`: SSL 설정의 활성화 여부를 설정한다. 기본값은 false

#### max connection과 max thread간의 관계

BIO(Blocking I/O)를 사용했을 때는, max-connctions가 max threads보다 많더라도 실제로는 max-thread 개수 만큼의 연결만 맺어진다.([톰캣 공식문서](https://tomcat.apache.org/tomcat-7.0-doc/config/http.html)) connection이 thread 개수가 많이 들어오는 경우 연결 대기상태가 되기 때문이다.

BIO에서 Executor를 사용하는 경우에는 기본값은 Executor의 maxThreads가 실제 max-connctions수가 된다. Executor는 기본적으로 스레드 풀을 사용하여 처리하기 때문이다.

반면, NIO(New I/O)를 사용할 때는 max threads보다 max-connctions가 많을 경우 max thread 개수를 초과하는 연결도 모두 받아들인다. NIO Connector에서는 BIO와 달리, 연결이 발생할 때 바로 새로운 Thread를 할당하지 않고(Connection : Thread ≠ 1:1) Poller라는 개념의 Thread에게 Connection(Channel)을 넘겨준다. Poller는 Socket들을 캐시로 들고 있다가 해당 Socket에서 data에 대한 처리가 가능한 순간에만 thread를 할당하기 때문에 thread 개수보다 많은 Connection을 관리할 수 있다.

{% embed url="https://github.com/genier1/genier1.gitbook.io/assets/19471818/74efc852-f8fa-4662-b72a-fb58f685f92c" %}

#### 적정 스레드 풀 개수는 어느정도가 좋을까?

어플리케이션마다 적정 스레드 풀 개수는 다를수 밖에 없다. 따라서 성능테스트를 통해 적정 스레드 풀 개수를 정하는 것이 가장 좋다.

일반적인 방식으로는 아래의 두가지 방식을 참고할 수 있다.

#### Little’s law

`L = λ * W`

* L - 동시에 처리되는 요청 수
* λ – long-term average arrival rate (RPS)
* W – the average time to handle the request (latency) λ - 장기 평균 도착률 (RPS, Requests Per Second) W - 요청 처리 평균 시간 (지연 시간)

예를 들어, 초당 1000개의 요청을 받는 애플리케이션이 있고, 각 요청을 처리하는데 평균 0.5초가 소요된다고 해보자.

Little’s law에 따르면 RPS = 1000, Latency = 0.5이므로 적절한 스레드풀 개수는 500개이다.

#### Brian Goetz

자바 병렬 프로그래밍의 저자 Brian Goetz는 다음과 같은 공식을 제안했다.

`Number of threads = Number of Available Cores * (1 + Wait time / Service time)`

* **Waiting time** - I/O 작업이 완료될 때까지 기다리는 시간
  * I/O 작업 뿐 아니라 모니터 락을 얻기 위해 기다리는 시간이나 스레드가 WAITING/TIMED\_WAITING 상태일 때의 시간도 포함될 수 있다.
* **Service time** - 실제로 작업을 수행하는 데 소요되는 시간
  * 예를 들어, HTTP 응답을 처리하거나, 마샬링/언마샬링하거나, 기타 변환 작업 등을 수행하는 시간

예를 들어, 4코어 CPU에서 실행되는 애플리케이션이 있다고 하자. 그리고 각 요청은 평균 0.5초 동안 외부 서비스로부터 HTTP 응답을 기다린다고 하자. 그리고 받은 HTTP 응답을 처리하고, 마샬링/언마샬링하며, 기타 변환 작업을 수행하는 데 평균 0.3초 정도 소요된다고 했다고 하자.

Core 개수 = 4, Wait time = 0.5, Service time = 0.3이므로 적절한 스레드 개수는 11개 이다. (4 \* (1 + 0.5 / 0.3))

#### 참고링크

* [https://sihyung92.oopy.io/spring/1](https://sihyung92.oopy.io/spring/1)
* [https://junuuu.tistory.com/799](https://junuuu.tistory.com/799)
* [https://thelogiclooms.medium.com/how-to-get-ideal-thread-pool-size-9b2c0bb74906](https://thelogiclooms.medium.com/how-to-get-ideal-thread-pool-size-9b2c0bb74906)
* [https://engineering.zalando.com/posts/2019/04/how-to-set-an-ideal-thread-pool-size.html](https://engineering.zalando.com/posts/2019/04/how-to-set-an-ideal-thread-pool-size.html)
* [https://wisdom-and-record.tistory.com/140](https://wisdom-and-record.tistory.com/140)
