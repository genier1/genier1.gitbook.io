# Spring AOP, CGLIB, JDK Proxy

#### What is AOP

AOP란 Aspect-oriented Programming(관점지향 프로그래밍)의 약자이다. 스프링 공식문서에서는 AOP를 OOP(Object-oriented Programming)의 보완재이며 스프링의 핵심 구성 요소 중 하나로 보고 있다.

스프링 공식문서에서는 Aspect가 여러 유형과 객체를 가로지르는 관심사(cross-cutting concerns)의 모듈화를 가능하게 한다고 말한다. 즉, AOP를 사용하면 어플리케이션 로직에서 핵심 기능과 부가 기능으로 관점을 나누고 이에 대해 모듈화가 가능해진다.

예를 들어, AOP의 대표적인 예시인 `@Transactional` 어노테이션을 보자. `@Transactional`이 없었다면 서비스단의 비즈니스 로직마다 DB 접근 로직.(커넥션 관리, 커밋, 롤백 등등)을 추가해야한다. 만약 DB 관련 로직에 수정이 발생한다면 해당 로직이 들어 있는 수십/수백개의 로직을 수정해야 할 것이다.

그러나 AOP를 사용하면 DB 접근 로직이라는 부가기능을 모듈화할 수 있다. 이를 Aspect라고 부른다. OOP에서 모듈화의 핵심 단위가 클래스라면, AOP에서의 모듈화 핵심 단위는 Aspect이다.

Aspect는 어플리케이션 핵심 로직과 분리하여 재사용할 수 있다. DB접근 로직 Aspect로 모듈화한다면 해당 Aspect에 수정이 발생하더라도 수십/수백개의 로직을 수정할 필요 없이 DB접근 로직만 수정하면 된다.

![images.jpeg](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d62cc179-6601-4921-8b18-7e018781fb4e/images.jpeg)

#### AOP의 구성 요소

* Aspect

횡단관심사(cross-cutting concerns)를 묶어서 모듈화 한 것을 말한다. 하나의 Aspect에는 Advice와 PointCut이 포함된다.

* Target

Aspect를 적용하는 곳 (클래스, 메서드 등등)

* Join Point

메소드 실행 또는 예외 처리와 같이 프로그램이 실행되는 지점을 말한다. Spring AOP에서 Join Point는 항상 메서드 실행을 의미한다.

* Advice

Join Point에서 Aspect에 의해 실행되는 동작이 담겨있다.

* PointCut

Join Point에 대한 상세한 스펙을 정의한다. Advice가 어느 시점에 실행될지는 PointCut을 통해 명시한다.

#### 스프링 AOP 실행원리

스프링 AOP는 기본적으로 프록시를 사용한다. 따라서 프록시를 통해 메서드를 실행하는 시점에만 AOP가 적용된다. 또한 스프링 AOP는 스프링 빈에만 AOP를 적용할 수 있다.

Spring framework 에서는 JDK dynamic proxy를 기본으로 사용하지만 Spring Boot에서는 CGLIB proxy를 Default로 사용한다. Spring boot에서 CGLIB를 디폴트로 채택한 이유는 뭘까. 리플렉션을 사용하는 JDK Dynamic proxy에 비해 unexpected cast exception이 발생 가능성이 적기 때문이라고 한다.

#### 스프링 AOP 구현

AOP를 어떻게 구현하는지 간단히 살펴보자. AOP를 구현하는 여러가지 옵션들이 있지만 우선 Spring AOP를 이용해 특정 어노테이션이 추가된 메서드에 로그를 찍어보자.

```sql
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Logging {
}
```

우선 메서드에 AOP를 적용하기 위한 Annotation을 만든다.

```sql
@Service
@Slf4j
public class OrderService {

    @Logging
    public void order(String product) {
        log.info("{} is ordered", product);
    }
}
```

AOP를 적용할 서비스와 서비스 하위의 메서드를 추가한다. String을 받아서 로그를 출력하는 메서드이다.

```java
@Aspect
@Component
@Slf4j
public class LogAspect {

    @Around("@annotation(Logging)")
    Object insertLog(ProceedingJoinPoint joinPoint) throws Throwable {
        log.warn("Log Start - {}", joinPoint.getSignature());
        return joinPoint.proceed();
    }
}
```

여기가 핵심이다. `@Aspect`를 통해 해당 클래스에서 AOP를 적용하는 것을 알 수 있다.

insertLog 메서드를 살펴보면 `@Around`를 통해 AOP를 적용할 범위를 지정하고 메서드 내부에 로그를 출력하는 기능을 추가한다. `@Around("@annotation(Logging)")` 이 아닌 `@Around("execution(* com.example.blog.service..*(..))")` 을 통해서 실행 범위를 지정하는 것도 가능하다. 우선은 많이 쓰는 어노테이션 기반으로 지정해보았다.

AOP를 적용하는 범위는 `@Around`외에도 `@Advice`와 `@PointCut`을 통해 더 세밀하게 조절할 수 있지만, 우선은 구현에 집중하기 위해 `@Around`를 사용한다.

```sql
@SpringBootTest
class OrderServiceTest {
    @Autowired
    OrderService orderService;

    @Test
    void loggingTest() throws Exception {
        orderService.order("Book");
    }
}
```

테스트를 실행하면 아래의 로그를 통해 Aspect가 잘 적용된 것을 확인할 수 있다.

```java
2022-11-29 20:27:30.741  INFO 37246 --- [    Test worker] com.example.blog.aop.LogAspect           : aop test2 - void com.example.blog.service.OrderService.order(String)
2022-11-29 20:27:30.749  INFO 37246 --- [    Test worker] com.example.blog.service.OrderService    : Book is ordered
```
