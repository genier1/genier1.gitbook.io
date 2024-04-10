# Reddison으로 분산락 구현하기

자바에서 레디스를 통해 분산락을 구현하는 방식으로 많이 사용하는 Redisson과 Lettuce를 비교해보자

|           | Redisson          | Lettuce   |
| --------- | ----------------- | --------- |
| 분산락 지원여부  | O (Lock 인터페이스 제공) | X (직접 구현) |
| 분산락 구현 방식 | Pub/Sub           | SpinLock  |

Redisson의 경우 Lock 인터페이스를 제공하는 특징이 있고 디폴트로 스핀락이 아닌 Pub/Sub 방식을 사용한다.

스핀락은 락 대기시간이 짧을 경우 ContextSwitching이 없는 스핀락 방식이 Pub/Sub보다 빠를 수 있다는 장점이 있지만, 대기 중인 스레드가 공유 자원의 상태를 무한 루프를 이용해 확인하는 방식이므로 스레드가 블록되며, CPU에 과부하를 줄 수 있다는 단점이 있다.

Redisson 문서에서는 짧은 시간에 수천개 이상의 락을 pub/sub 방식으로 처리할 경우, 네트워크 처리량 한계에 도달하고 Redis CPU 과부하가 발생할 수 있다고 한다. 이럴때 지수적 Backoff를 사용하는 스핀락을 통해 처리량을 줄일 수 있다. ([링크](https://github.com/redisson/redisson/wiki/8.-Distributed-locks-and-synchronizers#89-spin-lock))

> Thousands or more locks acquired/released per short time interval may cause reaching of network throughput limit and Redis CPU overload because of pubsub usage in Lock object. This occurs due to nature of Redis pubsub - messages are distributed to all nodes in Redis cluster. Spin Lock uses Exponential Backoff strategy by default for lock acquisition instead of pubsub channel.

#### Redisson의 Lock 동작 방식

Redisson은 default 설정으로 RedissonLock을 사용하며 RedissonLock은 아래와 같은 방식으로 동작한다.

1. tryLock을 통해 락 획득시도
2. waitTime 동안 락을 획득하지 못하면 false를 반환
3. threadId를 이용해 subscribe를 하여 락 획득.
4. 최종적으로 unsubscribe를 통해 락 해제

#### Redisson 소스코드 분석

RedissonLock 사용할 경우 pub/sub 방식으로 동작한다. 소스코드를 확인하면 this.subscribe, this.unsubscribe 함수를 통해 동작하는 것을 확인할 수 있다.

```java
// org.redisson.RedissonLock 클래스 
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
        long time = unit.toMillis(waitTime);
        long current = System.currentTimeMillis();
        long threadId = Thread.currentThread().getId();
        Long ttl = this.tryAcquire(waitTime, leaseTime, unit, threadId);
        if (ttl == null) {
            return true;
        } else {
            time -= System.currentTimeMillis() - current;
            if (time <= 0L) {
                this.acquireFailed(waitTime, unit, threadId);
                return false;
            } else {
                current = System.currentTimeMillis();
                CompletableFuture<RedissonLockEntry> subscribeFuture = this.subscribe(threadId);

                try {
                    subscribeFuture.toCompletableFuture().get(time, TimeUnit.MILLISECONDS);
                } catch (TimeoutException | ExecutionException var20) {
                    if (!subscribeFuture.cancel(false)) {
                        subscribeFuture.whenComplete((res, ex) -> {
                            if (ex == null) {
                                this.unsubscribe(res, threadId);
                            }

                        });
                    }
      ... 이하 생략
```

스핀락 방식을 사용하려면 RedissonSpinLock을 사용할 수 있다. RedissonSpinLock 클래스의 경우 tryLock 함수가 아래처럼 스핀락 방식으로 동작한다.

```java
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
        long time = unit.toMillis(waitTime);
        long current = System.currentTimeMillis();
        long threadId = Thread.currentThread().getId();
        Long ttl = this.tryAcquire(leaseTime, unit, threadId);
        if (ttl == null) {
            return true;
        } else {
            time -= System.currentTimeMillis() - current;
            if (time <= 0L) {
                this.acquireFailed(waitTime, unit, threadId);
                return false;
            } else {
                LockOptions.BackOffPolicy backOffPolicy = this.backOff.create();

                do {
                    if (ttl == null) {
                        return true;
                    }

                    current = System.currentTimeMillis();
                    Thread.sleep(backOffPolicy.getNextSleepPeriod());
                    ttl = this.tryAcquire(leaseTime, unit, threadId);
                    time -= System.currentTimeMillis() - current;
                } while(time > 0L);

                this.acquireFailed(waitTime, unit, threadId);
                return false;
            }
        }
    }
```

#### Redis 락 방식

Redis에서는

### 분산락 구현

쿠폰을 차감하는 동작을 분산락을 통해 구현해보자. Coupon이라는 엔티티가 있고, 해당 엔티티에는 availableStock 필드를 통해 수량을 표시한다.

```java
@Getter @ToString
@EqualsAndHashCode
@Entity(name = "coupon")
@NoArgsConstructor(access = PROTECTED)
public class Coupon {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private Long availableStock;

    public Coupon(String name, Long availableStock) {
        this.name = name;
        this.availableStock = availableStock;
    }

    public void decrease() {
        validateStockCount();
        this.availableStock--;
    }

    private void validateStockCount() {
        if (availableStock < 1) {
            throw new IllegalArgumentException();
        }
    }
}

```

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class CouponService {
    private final CouponRepository couponRepository;
    private final RedissonClient redissonClient;

    @Transactional
    public void couponDecrease(Long couponId) {
        String lockKey = "COUPON_DECREASE_" + couponId;
        RLock rLock = redissonClient.getLock(lockKey);

        try {
            boolean available = rLock.tryLock(5L, 3L, TimeUnit.SECONDS);
            if (!available) {
                throw new RuntimeException("redisson getLock fail.");
            }

            Coupon coupon = couponRepository.findById(couponId)
                    .orElseThrow();
            coupon.decrease();
        } catch (InterruptedException e) {
            throw new RuntimeException();
        } finally {
            TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
                @Override
                public void afterCompletion(int status) {
                    if (rLock.isHeldByCurrentThread()) {
                        rLock.unlock();
                    }
                }
            });
        }
    }
}
```

* 쿠폰을 차감하기 전에 redissonClient.getLock을 통해 락을 획득한다.
* 5초 동안 락을 획득하지 못하면 에러를 반환한다.
* 락을 획득하면 쿠폰을 조회한 후 해당 쿠폰의 수량을 차감한다.
* 해당 트랜잭션의 커밋이 완료되고 나면 락을 해제한다.
  * 커밋이 완료되기전 락이 해제되면 동시성 문제가 발생할 수 있다.

마켓컬리 기술 블로그에 분산락을 AOP를 통해 해결한 [좋은 글](https://helloworld.kurly.com/blog/distributed-redisson-lock)이 있으니 참고하면 좋다.

#### 테스트 코드

```java
@SpringBootTest
class CouponServiceTest {
    @Autowired
    CouponService couponService;
    @Autowired
    CouponRepository couponRepository;

    @Test
    void couponDecrease() throws Exception {
        Coupon coupon = new Coupon("COUPON", 100L);
        couponRepository.save(coupon);

        int couponCount = 100;
        CountDownLatch latch = new CountDownLatch(couponCount);
        ExecutorService executorService = Executors.newFixedThreadPool(couponCount);
        for (int i = 0; i < couponCount; i++) {
            executorService.submit(() -> {
               try {
                   couponService.couponDecrease(coupon.getId());
               } finally {
                   latch.countDown();
               }
            });
        }

        latch.await();
        executorService.shutdown();

        Coupon coupon1 = couponRepository.findById(coupon.getId())
                .orElseThrow();
        Assertions.assertThat(coupon1.getAvailableStock()).isZero();
    }
}
```



#### 참고링크

* [https://www.baeldung.com/redis-redisson](https://www.baeldung.com/redis-redisson)
* [https://hyperconnect.github.io/2019/11/15/redis-distributed-lock-1.html](https://hyperconnect.github.io/2019/11/15/redis-distributed-lock-1.html)
* [https://helloworld.kurly.com/blog/distributed-redisson-lock](https://helloworld.kurly.com/blog/distributed-redisson-lock)
* [https://github.com/redisson/redisson/wiki/8.-Distributed-locks-and-synchronizers](https://github.com/redisson/redisson/wiki/8.-Distributed-locks-and-synchronizers)
* [https://mangkyu.tistory.com/311](https://mangkyu.tistory.com/311)
* [https://0soo.tistory.com/m/256](https://0soo.tistory.com/m/256)
* [https://incheol-jung.gitbook.io/docs/q-and-a/spring/redisson-trylock](https://incheol-jung.gitbook.io/docs/q-and-a/spring/redisson-trylock)
