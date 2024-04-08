# MultipleBagFetchException in Hibernate

JPA에서 N+1 문제를 해결하기 위해 지연로딩으로 설정되어 있는 연관관계를 FetchJoin을 사용해 한 번에 호출해야 하는 경우들이 있다. 이 과정에서 **`MultipleBagFetchException`** 을 마주쳤던 상황과 이에 대한 해결 과정을 간략히 정리하고자 한다.

```java
@Entity
public class School {
    @Id
    @GeneratedValue
    @Column(name = "school_id")
    private Long id;
    private String name;

    @OneToMany(mappedBy = "school")
    private List<Professor> professors;
    @OneToMany(mappedBy = "school")
    private List<Student> students;
}
```

위와 같은 School 엔티티를 구성하고 연관된 엔티티를 한 번에 가져와보자. 편의상 JPQL을 사용해서 fetch join으로 연관된 엔티티인 `Professor`와 `Student`를 즉시 로딩으로 가져오는 쿼리를 사용했다.

```java
@Test
void multiBagExceptionTest() {
    IllegalArgumentException exception = Assertions.assertThrows(IllegalArgumentException.class, () -> {
        String jpql = "SELECT school FROM School school "
                + "JOIN FETCH school.students "
                + "JOIN FETCH school.professors ";
        em.createQuery(jpql);
    });

    System.out.println("exception.getMessage() = " + exception.getMessage());
}
```

해당 쿼리를 실행하니 `MultipleBagFetchException`의 근본 원인인 `IllegalArgumentException`을 뱉어낸다. 에러 메시지를 확인해보면 `org.hibernate.loader.MultipleBagFetchException: cannot simultaneously fetch multiple bags`라는 메시지를 출력한다.

#### Bag이란?

Bag이란 [하이버네이트](https://docs.jboss.org/hibernate/orm/5.4/userguide/html\_single/Hibernate\_User\_Guide.html)에서 사용하는 용어이며 자바 컬렉션에서는 사용하지 않는 용어이다. Bag은 `java.util.List`을 사용하며 순서가 없고 중복이 가능한 컬렉션을 의미한다.

문제는 엔티티에서 두개 이상의 Bag을 동시에 가져올 때 카타시안 곱을 만들 수 있는데, Bag 자체는 순서가 없으므로 하이버네이트에서 해당 Bag을 어떤 칼럼에 매핑할지 알 수가 없다. 따라서 이 경우에 하이버네이트에서는 `MultipleBagFetchException`을 반환한다. **결국, 이 문제를 해결하기 위해서는 쿼리 실행 시 중복된 Bag을 가져오지 않도록 해야한다.**

#### 해결책 1. List를 Set으로 변경

첫번째 방법은 List를 Set으로 변경하는 것이다. 서두에서 Bag이란 중복이 가능한 컬렉션이며 순서가 없으므로,두 개 이상의 Bag을 동시에 가져올 때 중복된 것이 있으면 문제가 발생한다고 말한 바 있다. Set은 List와 달리 하이버네이트에서 Bag으로 인식하지 않으므로 List를 Set으로 바꿔주면 Bag 타입이 1개만 있게 된다. 따라서 중복된 Bag으로 인한 \*\*`MultipleBagFetchException`\*\*을 방지할 수 있다.

```java
@Entity
public class School {
    @Id
    @GeneratedValue
    @Column(name = "school_id")
    private Long id;
    private String name;

    @OneToMany(mappedBy = "schoolName")
    private Set<Professor> professors;
    @OneToMany(mappedBy = "school")
    private List<Student> students;
}
```

기존에 List\<Professor>로 되어있던 타입을 Set\<Professor>로 수정한 후 아래의 테스트를 수행하면 `MultipleBagFetchException` 없이 정상적으로 코드가 동작하는 것을 확인할 수 있다.

```java
@Test
void multiBagExceptionTest() {
    String jpql = "SELECT school FROM School school "
                + "JOIN FETCH school.students "
                + "JOIN FETCH school.professors ";
    em.createQuery(jpql);
}
```

#### 해결책 2. JPQL 쿼리

필자의 경우에는 이미 해당 코드가 프로덕션에서 돌아가는 상황에서 엔티티를 함부로 수정할 수 없었다. 그래서 JPQL 쿼리를 여러번 날려서 가져오는 방식을 선택했다.

```java
@Test
void queryTest() {
    String jpql = "SELECT school FROM School school "
            + "JOIN FETCH school.students ";
    List<School> schools = entityManager.createQuery(jpql, School.class)
            .setHint(QueryHints.HINT_PASS_DISTINCT_THROUGH, false)
            .getResultList();

    String jpql2 = "SELECT school FROM School school "
            + "JOIN FETCH school.professors "
            + "WHERE school IN :schools ";

    List<School> schoolList = entityManager.createQuery(jpql2, School.class)
            .setParameter("schools", schools)
            .setHint(QueryHints.HINT_PASS_DISTINCT_THROUGH, false)
            .getResultList();
}
```

먼저 School에서 students 리스트를 가져오는 쿼리를 수행해서 schools를 가져온다. 그 다음 schools를 이용해 School에서 professor 리스트를 가져오는 쿼리를 수행하면, N+1 문제를 해결하면서 `MultipleBagFetchException`도 방지할 수 있다.

하이버네이트의 `default_batch_fetch_size`를 조절해서 해결하는 방법도 있다. 이 부분과 관련해서는 [청천향로님의 글](https://jojoldu.tistory.com/457)을 참고하면 좋다.

#### 해결책 3. @OrderColumn

같은 List라고 하더라도 Bag이 아닌 List로 인식하도록 만들 수 있다. `@OrderColumn`을 사용하면 해당 칼럼에 대해서는 데이터베이스에 순서값을 저장해서 사용한다는 의미이다.

위에서 Bag은 순서가 없고 중복이 가능한 컬렉션을 의미한다고 말한 바 있다. `@OrderColumn`을 사용하면 순서가 있는 컬렉션이 되므로 하이버네이트에서 Bag이 아닌 리스트로 인식한다.

```java
@Entity
public class School {
    @Id
    @GeneratedValue
    @Column(name = "school_id")
    private Long id;
    private String name;

    @OneToMany(mappedBy = "schoolName")
    private List<Professor> professors;
    @OneToMany(mappedBy = "school")
    @OrderColumn(name = "arrangement_index")
    private List<Student> students;
}
```

`students`필드에 `@OrderColumn`어노테이션을 붙이고 맨 처음에 `MultipleBagFetchException`이 발생했던 테스트 코드를 다시 실행하면 Exception이 발생하지 않아서 에러가 나는 것을 확인할 수 있다. 그러나 `@OrderColumn` 사용 시 주의해야할 점들이 몇가지 있으므로 신중하게 사용하자.

#### 여담

최근 회사에서 특정 API의 성능을 개선해야 하는 상황이 있었다. API 호출 시, 대량의 데이터를 조회해야 했고 해당 Entity내에는 @OneToMany로 매핑되어 있는 다른 엔티티가 6개가 있었다.

N+1문제가 발생하면서 해당 엔티티 조회 시 연관된 엔티티를 함께 조회하는 쿼리가 6번이 더 나가서 row 한 개당 7번의 쿼리가 나가고 있었다. 다시 말해, 해당 엔티티 1000개짜리 리스트를 조회하면 무려 7000개의 쿼리가 발생했던 것이다. 1000개가 아니고 10만개라면..?

결국 JPQL 쿼리의 In을 통해 기존 API 대비 호출시간을 1/10로 단축하는 의미있는 결과가 나왔고, 이 과정에서 JPA에 대해 좀 더 깊게 이해하게 된 좋은 경험이었다.

#### 출처

* [https://www.baeldung.com/java-hibernate-multiplebagfetchexception](https://www.baeldung.com/java-hibernate-multiplebagfetchexception)
