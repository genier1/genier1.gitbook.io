# JPA 더티체킹과 @dynamicupdate

최근 원인을 파악하기 힘든 이슈가 하나 있었다. 엔티티에서 Json으로 되어있는 필드가 갑자기 업데이트 된다는 것이었다. 중요한 점은 해당 필드를 업데이트 하는 코드는 하나도 없었다는 것이다.

이 문제를 해결하기 위해서는 JPA의 더티체킹 원리에 대해서 먼저 이해해야 한다.

#### JPA 더티체킹 원리

JPA의 구현체인 Hibernate에서는 트랜잭션 범위내에서 Entity를 조회할 경우 조회시점의 Entity 복사본을 만들어 둔다. (단, `@Transaction(readOnly=true)`로 설정한 경우에는 복사본을 만들지 않는다.)

그리고 트랜잭션이 끝날 때, 원본 Entity와 복사본을 비교하여 원본 Entity에 변경이 있다면 업데이트 쿼리를 수행한다. 더티체킹을 통해 생성되는 업데이트 쿼리는 기본적으로 모든 필드를 업데이트한다.

왜 모든 필드를 업데이트 할까? 두가지 장점이 있다.

* 생성되는 쿼리가 같아 **부트 실행시점에 미리 만들어서 재사용**가능합니다.
* 데이터베이스 입장에서 쿼리 재사용이 가능하다
  * 동일한 쿼리를 받으면 이전에 파싱된 쿼리를 재사용한다

건드리지도 않았던 Json 필드가 갑자기 업데이트 된 원인은 바로 여기에 있었다.

#### 문제의 원인

JPA에서 특정 필드를 업데이트할 경우 더티체킹에 의해 모든 필드가 업데이트 되었고 이 과정에서 Json 필드에 대한 컨버터가 동작하면서 해당 Json 필드가 컨버터에 의해 업데이트 되버린 것이었다.

따라서 이를 전혀 의도하지 않았던 입장에서는 버그가 발생했다고 생각했던 것이다.

```java
@Getter @Setter
@Table(name = "cars")
@Entity
public class Car {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String brand;

    public Car(String name, String brand) {
        this.name = name;
        this.brand = brand;
    }
}
```

먼저 Car 엔티티를 생성한다.

```java
@Service
@RequiredArgsConstructor
public class CarService {
    private final CarRepository carRepository;

    @Transactional
    public void updateName(Long id, String name) {
        Optional<Car> car = carRepository.findById(id);
        car.ifPresent(i -> i.setName(name));
    }
}
```

그리고 업데이트를 위한 서비스 클래스를 하나 생성한다.

```java
@SpringBootTest
@Transactional
class CarTest {
    @Autowired
    CarRepository carRepository;
    @Autowired
    CarService carService;
    @Autowired
    EntityManager em;

    @Test
    void updateQueryTest() {
        Car sonata = carRepository.save(new Car("소나타", "현대"));
        em.flush();
        
        carService.updateName(sonata.getId(), "제네시스");
        em.flush();
    }
```

위와 같은 테스트 코드를 만들어서 실행하면, 아래와 같은 SQL 문이 실행된다.

```sql
# carRepository.save 실행 시 수행
insert into cars
(brand, name) 
values (?, ?)

# carService.updateName 실행 시 수행
update cars 
set brand=?, name=? 
where id=?
```

예상과 다르게 name만 변경되는 것이 아니라, brand까지 update문에 포함 된 것을 확인할 수 있다.

#### 전체 필드가 업데이트 되는 이유

JPA에서는 전체 필드를 업데이트하는 방식을 기본값으로 사용한다. 전체 필드를 업데이트하는 방식의 장점은 다음과 같다.

* 생성되는 쿼리가 같아 부트 실행시점에 미리 만들어서 재사용이 가능
* 데이터베이스 입장에서 쿼리 재사용이 가능

평상시에는 이런 상황이 문제 없지만 전체 필드가 2\~30개 이상이 되어 성능상의 이유로 인해 특정 필드만 업데이트 하거나, 업데이트 되기를 원하지 않는 필드가 전체 업데이트로 인해 함께 업데이트되는 상황을 방지하고 싶을 수 있다.

#### 원하는 필드만 업데이트 하기

`@DynamicUpdate`를 사용하면 업데이트 한 필드에 대해서만 Update 쿼리가 수행되도록 할 수 있다.

```java
@Table(name = "cars")
@Entity
@DynamicUpdate 
public class Car {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String brand;

    public Car(String name, String brand) {
        this.name = name;
        this.brand = brand;
    }
}
```

Car 객체에 대해 `@DynamicUpdate` 어노테이션을 추가한 후, 맨 처음에 수행했던 테스트를 다시 실행한 후 수행되는 쿼리를 살펴보자.

```sql
@Test
void updateQueryTest() {
    Car sonata = carRepository.save(new Car("소나타", "현대"));
    em.flush();
    
    carService.updateName(sonata.getId(), "제네시스");
    em.flush();
}
```

sql문을 확인해보니 원했던 것 처럼 업데이트 되는 필드에 대해서만 update 쿼리가 나간 것을 확인할 수 있다.

```sql
update cars 
set name=? 
where id=?
```

#### 출처

* [JPA Entity Select에서 Update 쿼리 발생할 경우](https://jojoldu.tistory.com/536?category=637935)
* [@DynamicUpdate with Spring Data JPA](https://www.baeldung.com/spring-data-jpa-dynamicupdate)
