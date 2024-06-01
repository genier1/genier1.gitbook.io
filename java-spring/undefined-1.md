# 서킷브레이커

서킷 브레이커는 호출된 서비스의 상태를 모니터링하고, 일정한 실패 비율이나 지연 시간을 초과하는 호출이 발생할 경우 해당 서비스 호출을 차단하여 장애가 전파되는 것을 방지하는 기술이다.

지금부터 설명하는 서킷브레이커 관련 설명은 resilience4j 기준이다.

{% embed url="https://github.com/genier1/genier1.gitbook.io/assets/19471818/edc5966a-02a3-4bee-85d6-f3da8142ddd1" %}

서킷브레이커는 3가지의 보통 상태(OPEN, CLOSED, HALF\_OPEN)와 3가지의 특별한 상태(DISABLED, FORCED\_OPEN, METRICS\_ONLY)를 가진다.

* CLOSED
  * 서킷브레이커가 닫혀있는 상태이며, 서킷브레이커가 감싼 내부 프로세스가 요청과 응답을 정상적으로 주고 받는다.
* OPEN
  * 장애로 인해 서킷브레이커가 열린 상태이다. 에러를 반환하거나 지정한 callback 메소드를 호출한다.
* HALF\_OPEN
  * fall back 응답을 수행하고 있지만 실패율을 측정해서 CLOSE 또는 OPEN 으로 변경될 수 있는 상태이다.

아래의 세가지 특수 상태도 있다.

* DISABLED
  * 서킷브레이커를 강제로 CLOSED 한 상태. 메트릭을 수집하지 않고 상태 변화도 없다.
* FORCED\_OPEN
  * 서킷브레이커를 강제로 OPEN 한 상태. 메트릭을 수집하지 않고 상태 변화도 없다.
* METRICS\_ONLY
  * 메트릭을 수집하고 이벤트를 발행하지만, 다른 상태로는 전환되지 않는 상태이다.

위 상태값은 transitionTo\~ 메서드를 통해 임의로 조절할수도 있다.

```java
@Test
void transitionCircuitBreakerState() throws Exception {
    CircuitBreaker circuitBreaker = circuitBreakerRegistry
            .circuitBreaker("name", CircuitBreakerConfig.ofDefaults());

    Assertions.assertThat(circuitBreaker.getState()).isEqualTo(CLOSED);

    circuitBreaker.transitionToDisabledState();
    Assertions.assertThat(circuitBreaker.getState()).isEqualTo(DISABLED);

    circuitBreaker.transitionToForcedOpenState();
    Assertions.assertThat(circuitBreaker.getState()).isEqualTo(FORCED_OPEN);

    circuitBreaker.transitionToMetricsOnlyState();
    Assertions.assertThat(circuitBreaker.getState()).isEqualTo(METRICS_ONLY);
}
```

#### 종류

* COUNT\_BASED

```java
public class FixedSizeSlidingWindowMetrics implements Metrics {

    private final int windowSize;
    private final TotalAggregation totalAggregation;
    private final Measurement[] measurements;
    int headIndex;
}
```

Count-based 슬라이딩 윈도우는 N개의 측정값을 가진 순환 배열로 구현되어 있다.

새로운 호출 결과가 발생할때, totalAggregation을 점진적으로 업데이트 하며, windowSize를 초과하는 이전의 호출결과는 제거된다.

예를 들어, windowSize = 10일 때, 11번째 호출결과가 들어오면 첫번째 호출결과는 제거되고 totalAggregation는 2 \~ 10번째 호출 결과를 기준으로 작성된다.

* TIME\_BASED

```java
public class SlidingTimeWindowMetrics implements Metrics {

    final PartialAggregation[] partialAggregations;
    private final int timeWindowSizeInSeconds;
    private final TotalAggregation totalAggregation;
    private final Clock clock;
    int headIndex;
}

class AbstractAggregation {
    long totalDurationInMillis = 0L;
    int numberOfSlowCalls = 0;
    int numberOfSlowFailedCalls = 0;
    int numberOfFailedCalls = 0;
    int numberOfCalls = 0;
}
public class PartialAggregation extends AbstractAggregation {
    private long epochSecond;
}

```

Count-based 슬라이딩 윈도우는 N개의 PartialAggregation 순환 배열로 구성되어 있다.

만약 timeWindowSizeInSeconds가 10초라면 partialAggregations의 size는 10이다.

partialAggregations는 실패한 호출 수, 느린 호출 수, 전체 호출 수를 계산하며 epochSecond를 통해 전체 호출시간을 계산한다.

#### 주요 설정

circuitbreaker의 기본값과 설명은 다음과 같다.

```yaml
resilience4j:
 circuitbreaker:
   configs:
     default:
       failureRateThreshold: 50 # 실패율 임계값 백분율. 실패율이 임계값 이상이면 CircuitBreaker가 open되고 호출을 차단함.
       slowCallRateThreshold: 100 # 백분율 임계값. slowCallDurationThreshold보다 호출 시간이 길면 느린 호출로 간주. 느린 호출 비율이 임계값 이상이면 CircuitBreaker가 open되고 호출을 차단함.
       slowCallDurationThreshold: 60000 # 호출을 느리다고 판단하는 시간 임계값 (ms).
       permittedNumberOfCallsInHalfOpenState: 10 # half open 상태일 때 허용되는 호출 수.
       maxWaitDurationInHalfOpenState: 0 # CircuitBreaker가 Half Open 상태로 유지되는 최대 시간 (ms). 0이면 허용된 호출이 완료될 때까지 대기함.
       slidingWindowType: COUNT_BASED # 호출 결과를 기록하는 슬라이딩 창 유형. COUNT_BASED면 최근 slidingWindowSize 호출을, TIME_BASED면 최근 slidingWindowSize 초 동안의 호출을 기록함.
       slidingWindowSize: 100 # 호출 결과를 기록하는 슬라이딩 창 크기.
       minimumNumberOfCalls: 100 # CircuitBreaker가 오류율이나 느린 호출 비율을 계산하기 위해 필요한 최소 호출 수. 예를 들어 10이면 최소 10개 호출이 있어야 실패율 계산 가능.
       waitDurationInOpenState: 60000 # CircuitBreaker가 open에서 half-open으로 전환하기 전 대기 시간 (ms).
       automaticTransitionFromOpenToHalfOpenEnabled: false # true면 CircuitBreaker가 자동으로 open에서 half-open으로 전환됨. false면 대기 시간 후에도 호출이 있어야 전환됨.
       recordExceptions: # 실패로 기록될 예외 목록. 명시적으로 무시되지 않는 한 일치하는 예외는 실패로 간주.
       ignoreExceptions: # 무시될 예외 목록. 일치하는 예외는 실패나 성공으로 간주되지 않음.
       recordFailurePredicate: "throwable -> true" # 예외를 실패로 기록할지 판단하는 사용자 지정 Predicate. 기본값은 모든 예외를 실패로 기록.
       ignoreExceptionPredicate: "throwable -> false" # 예외를 무시할지 판단하는 사용자 지정 Predicate. 기본값은 예외를 무시하지 않음.
```

#### 사용 예시

CircuitBreaker의 사용법은 어노테이션 또는 CircuitBreakerRegistry를 통해 사용할 수 있다.

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    configs:
      default:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10
        failureRateThreshold: 50
        permittedNumberOfCallsInHalfOpenState: 5
```

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/circuit-breaker")
public class CircuitBreakerController {
    private final CircuitBreakerService circuitBreakerService;

    @GetMapping("/{id}")
    public String test(@PathVariable Long id) {
        return circuitBreakerService.getMessage(id);
    }
}

@Service
@RequiredArgsConstructor
public class CircuitBreakerService {

    @CircuitBreaker(name = "message-circuit-breaker", fallbackMethod = "defaultMessage")
    public String getMessage(Long id) {
        if (id < 10L) {
            return "message: " + id;
        }
        throw new RuntimeException();
    }

    public String defaultMessage(Long id, CallNotPermittedException exception) {
        return "defaultMessage";
    }
}
```

주의할 점은 fallbackMethod인 defaultMessage()의 인자에 `CallNotPermittedException` 을 추가해줄 경우, 최초에 slidingWindowSize 설정 만큼 에러를 반환한 후 서킷브레이커가 OPEN 상태가 되어야 fallback 메서드인 `defaultMessage()`를 호출한다는 점이다.

`Throwable`이나 Throwable을 상속하는 에러를 인자로 추가할 경우 에러가 발생하자마자 바로 `defaultMessage()`를 호출하므로 클라이언트에서 500 에러를 받을 일이 없다.

왜 이런 현상이 발생하는지 코드를 통해 살펴보자

```java
public class DefaultFallbackDecorator implements FallbackDecorator {

    @Override
    public boolean supports(Class<?> target) {
        return true;
    }

    @Override
    public CheckedSupplier<Object> decorate(FallbackMethod fallbackMethod,
                                            CheckedSupplier<Object> supplier) {
        return () -> {
            try {
                return supplier.get();
            } catch (IllegalReturnTypeException e) {
                throw e;
            } catch (Throwable throwable) {
                return fallbackMethod.fallback(throwable);
            }
        };
    }
}

```

getMessage에서 RuntimeException을 반환하게 되면`DefaultFallbackDecorator.decorate()`에서 fallbackMethod.fallback(throwable)을 호출하게 된다.

```java
@Nullable
public Object fallback(Throwable thrown) throws Throwable {
    if (fallbackMethods.size() == 1) {
        Map.Entry<Class<?>, Method> entry = fallbackMethods.entrySet().iterator().next();
        if (entry.getKey().isAssignableFrom(thrown.getClass())) {
            return invoke(entry.getValue(), thrown);
        } else {
            throw thrown;
        }
    }

    Method fallback = null;
    Class<?> thrownClass = thrown.getClass();
    while (fallback == null && thrownClass != Object.class) {
        fallback = fallbackMethods.get(thrownClass);
        thrownClass = thrownClass.getSuperclass();
    }

    if (fallback != null) {
        return invoke(fallback, thrown);
    } else {
        throw thrown;
    }
}
```

`*entry*.getKey().isAssignableFrom(*thrown*.getClass())` 이 True라면 `invoke(*entry*.getValue(), *thrown*)`를 통해 fallbackMethod를 호출하게 된다.

위의 예시에서 서킷브레이커로 감싸진 defaultMessage 메서드에서는 RuntimeException을 발생시킨다. 반면, fallback 메서드의 에러 타입은 `CallNotPermittedException` 이다.

따라서 `CallNotPermittedException.class.isAssignableFrom(RuntimeException.class)` 이 false이므로 fallback 메서드를 호출하지 않고 test 메서드에서 던진 RuntimeException을 그대로 반환하게 된다.

다시 말해서, 서킷브레이커로 감싸진 프로세스에서 발생할 수 있는 에러 클래스 타입과 fallback메서드에서 인자로 받는 에러 클래스 타입을 잘 확인해야 의도치 않은 동작을 방지할 수 있다.

#### RegistryEventConsumer

RegistryEventConsumer interface를 통해 CircuitBreaker의 상태변화 이벤트를 소비하고 별도의 동작을 지정할 수 있다.

```java
public class CircuitBreakerEventListener implements RegistryEventConsumer<CircuitBreaker> {

    @Override
    public void onEntryAddedEvent(EntryAddedEvent<CircuitBreaker> entryAddedEvent) {
        entryAddedEvent.getAddedEntry().getEventPublisher()
                .onFailureRateExceeded(event -> log.warn("{} failure rate {}%", event.getCircuitBreakerName(), event.getFailureRate())
                )
                .onError(event -> log.error("{} ERROR!!", event.getCircuitBreakerName())
                )
                .onStateTransition(
                        event -> log.info("{} state {} -> {}",
                                event.getCircuitBreakerName(), event.getStateTransition().getFromState(), event.getStateTransition().getToState())
                );
    }

    @Override
    public void onEntryRemovedEvent(EntryRemovedEvent<CircuitBreaker> entryRemoveEvent) {
        entryRemoveEvent.getRemovedEntry().getEventPublisher()
                .onStateTransition(event -> log.debug("onEntryRemovedEvent"));
    }

    @Override
    public void onEntryReplacedEvent(EntryReplacedEvent<CircuitBreaker> entryReplacedEvent) {
        entryReplacedEvent.getNewEntry().getEventPublisher()
                .onStateTransition(event -> log.debug("onEntryReplacedEvent"));
    }
}

```

#### 참고링크

* [https://hyeon9mak.github.io/spring-circuit-breaker](https://hyeon9mak.github.io/spring-circuit-breaker)
* [https://techblog.woowahan.com/15694/](https://techblog.woowahan.com/15694/)
* [https://resilience4j.readme.io/docs/circuitbreaker](https://resilience4j.readme.io/docs/circuitbreaker)
