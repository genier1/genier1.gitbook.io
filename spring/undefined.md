# 클래스 상속시 주의해야 할 점

### TL:DR

1. IS-A 관계가 아니라면 상속하지 말것
2. 클래스 상속 시 발생할 수 있는 문제를 고려할 것
   1. 결합도 증가
   2. 캡슐화 위반
   3. 클래스 폭발 및 중복 코드 양산
   4. 슈퍼클래스의 메소드 상속으로 인한 문제 발생
3. 클래스 상속보다는 인터페이스 상속이나 합성을 고려할 것

### 상속 시 주의사항

#### 결합도 상승

상위 클래스가 변경되면, 해당 클래스를 상속받은 클래스들까지 같이 변경되어야 할 수 있다.

아래의 예시를 통해 살펴보자.

```sql
public class Vehicle {
    private String color;
    private double price;

    public Vehicle(String color, double price) {
        this.color = color;
        this.price = price;
    }

    public double calculatePrice() {
        return 0.9 * price;
    }
}

public class Car extends Vehicle {
    private double tax;

    public Car(String color, double price, double tax) {
        super(color, price);
        this.tax = tax;
    }

    @Override
    public double calculatePrice() {
        double price = super.calculatePrice();
        return price + tax;
    }
}
```

Car 클래스는 Vehicle 클래스를 상속받고 있다.

Vehicle에 필드가 추가되어 생성자가 변경될 경우, Car 클래스의 생성자도 변경되어야 한다.

또한 Vehicle 클래스의 calculatePrice가 price에 0.9가 아닌 0.8이 곱해지는 것으로 변경된다면 Car 클래스의 calculatePrice 결과가 달라져버린다.

Vehicle 클래스를 상속받은 클래스가 수십개라면(오토바이, 자전거, 버스 등등), Vehicle 클래스의 사소한 변경도 수십개 클래스에 모두 영향을 미치는 결과를 초래한다.

#### 캡슐화 위반 가능성

캡슐화란 객체의 속성과 행위를 외부에서 접근할 수 없도록 하는 것이다. 이를 통해 실제 구현 내용의 일부를 감추는 효과를 얻게 된다.

그렇다면 상속이 왜 캡슐화를 위반할 수 있는것일까?

하위 클래스는 상위 클래스에서 제공하는 구현을 알고있어야 하기 때문이다. Car 클래스의 calculatePrice는 Vehicle의 calculatePrice 함수를 통해 가격을 받아온 후, 세금을 더하는 방식으로 동작한다. 즉, 상위 클래스인 Vehicle 클래스의 calculatePrice가 어떤 역할을 하는지 알고 있어야만 하위 클래스인 Car의 calculatePrice 구현을 완성할 수 있다.

#### 구조적인 결함 발생

자바의 Stack 클래스는 상속의 단점을 잘 보여주는 경우다.

```java
Stack<String> stack = new Stack<>();
stack.push("salad");
stack.push("poke");
stack.push("steak");

stack.add(0, "chicken");

// 실패, 실제로는 steak를 반환
assertEquals("chicken", stack.pop()); 
```

Stack은 LIFO 구조이므로 stack에서 pop을 했을때 “4th”가 나올 것으로 예상하겠지만 실제로는 그렇지 않다.

Vector를 상속받으면서 vector의 `add(int index, E element)` 메소드도 함께 상속받다보니, Stack 클래스에서도 특정 index에 element 삽입이 가능해져 버렸다.

상속을 받을 때 슈퍼클래스의 메서드를 모두 함께 상속받는 과정에서 이런 구조적인 결함이 발생할 수 있다. 해당 이유 뿐 아니라 여러가지 이유로 인해 Java에서는 Stack 대신 Deque 사용을 권장한다.

#### 클래스 폭발 및 가독성 저하

A → B → C 형태와 같이 클래스 상속이 두 단계만 되더라도 C 클래스가 어떤 필드와 메서드로 합성되어 있는지 알아보려면 A, B 클래스를 살펴보아야 하는 문제가 생긴다. A, B 클래스를 상속받은 클래스가 수십개라면 코드 자체를 이해하는 것이 매우 어려워 진다.

개인적으로는 두 단계 이상의 클래스 상속은 되도록 피하려고 한다.

### 그렇다면 어떻게 해야 할까

#### IS-A 관계일 경우에 상속을 사용하자.

많은 글에서 Is-A 관계일 때 상속을 사용하고, Has-A 관계일 때 Compisition(합성)을 사용하라고 권장한다.

Is-A 관계의 예시이다.

* 사람은 동물이다.

위와 같이 Is-A 관계일 경우에는 상속을 사용하는것을 권장한다. 단, 위에서 언급한 문제가 발생할 수 있는 점은 조심하자.

Has-A 관계의 예시이다.

* 자동차는 엔진을 가지고 있다.

위와 같이 Has-A 관계일 경우에는 상속보다는 합성을 사용하는 것이 좋다.

#### 합성(Composition) 사용

Stack 클래스에서 상속으로 발생한 문제를 해결하는 방법으로는 합성이 있다.

```java
public class Stack<E> {
    private final Vector<E> vector = new Vector<>();
 
    public E push(E item) {
        vector.add(item);
        return item;
    }
 
    public E pop() {
        return vector.remove(vector.size() - 1);
    }
 
    public E peek() {
        return vector.get(vector.size() - 1);
    }
 
    public boolean empty() {
        return vector.isEmpty();
    }
 
}
```

위와 같이 합성을 사용하게 되면 Vector 클래스의 메서드가 외부로 노출되지 않아 이전에 발생했던 문제가 해결된다.

#### 인터페이스 상속

아무리 잘 사용하더라도 클래스 상속은 여러 문제를 내포하고 있으므로, 가능하다면 extends를 통한 클래스 상속보다는 interface 상속을 고려해보자.

#### 참고링크

* [https://hoons-dev.tistory.com/106](https://hoons-dev.tistory.com/106)
* [https://incheol-jung.gitbook.io/docs/q-and-a/architecture/undefined-2](https://incheol-jung.gitbook.io/docs/q-and-a/architecture/undefined-2)
* [https://jake-seo-dev.tistory.com/404](https://jake-seo-dev.tistory.com/404)
* [https://colabear754.tistory.com/125](https://colabear754.tistory.com/125)
