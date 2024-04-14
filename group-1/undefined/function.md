---
description: 3 ~ 5강 정리
---

# Function

## Function

### TL:DR

* 함수는 작아야 한다.
* 이름을 잘 짓고, 함수를 작게 만들면 모든 사람들의 시간을 절약해준다.
  * 처음에는 정보가 없으므로 적절한 이름을 못지을 수 있다. 나중에라도 이름을 명확하게 지어라
* 함수는 한가지 일만 해야 한다.
  * 한가지 일만 하는지 확신할 수 있는 유일한 방법은 Extract Till You Drop이다.
* 함수는 한가지 일만 해야한다.
  * 함수가 한가지 일만 하는지 어떻게 확인할 수 있을까?
    * Extract Till You Drop
    * if, while 문 등에서 {} 가 보이면 extract 대상
* 4줄 이하로 작성해야함
  * 그러기 위해서는 indentation, while, nested if 등이 없어야 함
  * 잘 지어진 서술적인 이름을 갖는 많은/작은 함수들로 유지해야함
* 큰 함수를 보면 클래스로 추출할 생각을 해야 함
  * 큰 함수는 실제로는 클래스가 숨어 있는 곳이다.
  * 어느정도가 큰 함수일까? → 대략 20줄 이상이면 큰 함수이다.
  * Extract Method Object
  * 클래스는 일련의 변수들로 동작하는 기능의 집합
* 함수의 첫번째 규칙 - 함수는 작아질 수 있는 한 최대한 작아져야 한다.
  * 블록이 적어야한다
    * if, else, while 등의 내부 블록은 한줄이어야 함
  * indenting이 적어야 한다.
    * 함수는 중첩구조를 가질만큼 크면 안된다.
* 함수가 수행하는 모든일의 추상화 수준이 같다면 이 함수는 한가지 일을 하는 것이다.
  * 단순한 재인용이 아닌 이름으로 함수를 추출할 수 있을때까지 함수를 추출한다.

## Function Structure

* 함수의 인자가 많아지면 복잡도가 증가한다.
  * 최대 3개의 인자를 사용하라.
  * Introduce Parameter Object 활용([참고 링크](https://www.refactoring.com/catalog/introduceParameterObject.html))
* 생성자에 많은 수의 인자를 넘겨야 한다면 빌더 패턴 사용을 권장
* boolean 인자 사용 금지
  * 2가지 이상의 일을 하는 것이므로 차라리 함수 2개로 분리하라.
* Output 인자를 사용하지 마라.
  * 리턴 값이 있을 경우 인자가 변경되어 반환 되는것이라고는 생각하기 어렵다.
* Null 방어
  * null을 전달/기대하는 함수는 boolean을 전달하는 것 만큼 잘못된 것이다. → 2개의 함수로 분리하라
  * 방어적 프로그래밍 지양
    * 코드를 null, 에러 체크로 더럽히지 마라.
    * 이는 팀원과 단위 테스트를 못믿는다는 말이다.
    * null 여부를 지속적으로 조사하는게 아니라 단위 테스트에서 검증해야 한다.
  * 단, Public API인 경우에는 방어적으로 프로그래밍하라
*   객체지향의 장점

    * Polymophic 인터페이스를 삽입함으로써 Runtime 의존성은 그대로 둔 채 소스코드의 의존성을 역전시킨다.(DI)
    * 아래처럼 모듈 A가 모듈 B를 사용하는 경우 독립적 배포, 컴파일이 불가능



    <img src="https://github.com/genier1/genier1.gitbook.io/assets/19471818/7e37fb5d-ad7c-4a74-a617-cf5a363e7c47" alt="" data-size="original">

    *   아래처럼 모듈 A는 인터페이스에 의존하고 B는 인터페이스로부터 Derive 한다.

        ![](https://github.com/genier1/genier1.gitbook.io/assets/19471818/552a1ad8-7094-46dd-bb9a-046667391cb2)

        * Runtime 의존성은 그대로 둔 채로 소스코드의 의존성을 역전시켰음.
        * 독립적인 배포, 컴파일이 가능해짐, 테스트도 용이해짐


* Switch 문장 사용은 왜 꺼리는가?
  * 각 Case 문장이 외부 모듈에 의존성을 갖게됨
    * 5개 모듈에 의존하고 있다고 하면 그 중 하나만 변경되어도 영향을 받게 됨 (fan-out Problem)
  * Switch 문 제거 절차
    * switch 문장을 polymophic 인터페이스 호출로 전환
    * case에 있는 문장을 별도 클래스로 추출하여 변경 영향이 발생하지 않도록 한다.



### 참고 예제

[https://github.com/msbaek/fitness-example](https://github.com/msbaek/fitness-example)

[https://github.com/msbaek/print-prime](https://github.com/msbaek/print-prime)

[https://github.com/msbaek/videostore](https://github.com/msbaek/videostore)
