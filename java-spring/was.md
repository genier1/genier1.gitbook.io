# 서블릿, 서블릿컨테이너, WAS 정리

#### 서블릿이란

* 클라이언트 요청을 처리하고, 동적으로 결과를 반환하는 웹 프로그래밍 기술
* 이전의 CGI와는 달리, 스레드를 이용하여 동작한다.
* Java에서는 Servlet 인터페이스를 구현해서 사용

#### 서블릿 컨테이너

* 서블릿을 관리하는 용도
* 스프링 부트에서는 톰캣이 내장 서블릿 컨테이너의 역할을 한다.
* 서블릿 컨테이너의 주요 기능
  * 서블릿 생명주기 관리
  * 웹서버와 통신 지원
  * 멀티스레드 관리
  * 보안관리

#### 서블릿 생명주기

<figure><img src="https://github.com/genier1/genier1.gitbook.io/assets/19471818/423d0395-1425-4905-9d55-14583aabcab6" alt=""><figcaption></figcaption></figure>

* 서블릿이 메모리에 없으면 init()을 통해 서블릿 생성
* service() 메서드를 통해 GET, POST 등에 대한 응답 처리
* 서블릿 컨테이너가 서블릿에 종료요청을 하면 destory() 메서드를 통해 종료

#### Custom Servlet 만들기

[https://github.com/genier1/genier1.gitbook.io/assets/19471818/93bb80b9-06be-4085-a695-82a3e3211196](https://github.com/genier1/genier1.gitbook.io/assets/19471818/93bb80b9-06be-4085-a695-82a3e3211196)

스프링 MVC에서 사용하는 DispatcherServlet도 Servlet 인터페이스를 상속받는다. 특정 Url로 요청이 들어올 경우 특정 응답을 반환하도록 하는 Servlet을 만들어보자.

```java
@RestController
@RequestMapping("/servlet")
public class ServletController {

    @GetMapping("/test")
    public String test() {
        return "test";
    }
}
```

/servlet/test 로 요청이 들어오면 “test”라는 텍스트를 반환하도록 하는 컨트롤러가 있다. 해당 Url에 대해 응답이 들어올 경우 다른 응답을 반환하도록 하는 CustomServlet을 만들어보자.

```java
@WebServlet(urlPatterns = "/servlet/*")
public class CustomServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        response.setContentType("text/plain");
        response.getWriter().println("Custom Servlet Response");
    }
}
```

HttpServlet을 상속하여 doGet 메서드를 오버라이드 하면 urlPatterns에 포함되는 GET 메서드의 호출은 모두 해당 서블릿을 통과하게 된다.

GET 메서드 뿐 아니라 모든 메서드에 대해 같은 처리를 하고싶다면 service() 메서드를 오버라이딩 하면 된다.  HTTP Method 별로 다르게 처리하고 싶다면 doGet, doPost, doPut, doDelete 등의 메서드를 오버라이딩 하면 각각 다른 처리를 할 수 있다.

해당 코드에 따르면 /servlet/ 하위의 url의 GET 요청에 대해서는 무조건 “Custom Servlet Response” 라는 Text를 반환하게 된다.

그런데 코드를 작성한 후 어플리케이션을 실행해 API를 호출하면 여전히 “test”라는 응답을 받게 될 것이다. 추가 설정이 빠졌기 때문이다.

```java
@SpringBootApplication
@ServletComponentScan
public class BlogApplication {

	public static void main(String[] args) {
		SpringApplication.run(BlogApplication.class, args);
	}

//	// ServletComponentScan을 사용하지 않을 경우 아래 방식을 통해 빈 등록
//	@Bean
//	public ServletRegistrationBean customServletRegistrationBean(){
//		return new ServletRegistrationBean(new CustomServlet(), "/servlet/*");
//	}
}

```

설정은 두가지 방법이 있다.

1. `@ServletComponentScan` 을 통해 `@WebServlet` 이 있는 클래스들이 컴포넌트 스캔에 포함되도록 처리한다.
   1. 이 경우에는 @WebServlet을 선언해주어야 한다.
2. ServletRegistrationBean을 빈으로 등록해서 처리한다.
   1. Servlet 클래스에는 @WebServlet이 없어도 된다.

#### WAS와 Web Server

<figure><img src="https://github.com/genier1/genier1.gitbook.io/assets/19471818/df5a1c92-460b-4b1a-90c7-48b9e20172bc" alt=""><figcaption></figcaption></figure>

* Web Server
  * 주로 정적인 요청을 처리한다.
  * 동적인 요청이 올 경우 WAS로 보낸다.
  * 리버스프록시, 로드밸런싱 등을 처리하기도 한다.
  * Nginx, Apache 등이 대표적인 Web Server이다.
* WAS(Web Application Server)
  * DB조회나 다양한 비즈니스 로직 처리를 요구하는 **동적인 컨텐츠**를 제공하기 위한 Application Server
  * 웹 서버 + 웹 컨테이너를 포함한다.
  * Tomcat, JBoss 등이 대표적인 WAS이다.
* Web Server와 WAS를 왜 구분하는가.
  * WAS가 정적 컨텐츠 처리까지 모두 진행할 경우 부하가 커지는 문제가 있음
  * Web Server 하나에 여러 WAS를 연결해 로드밸런싱이나 reverse proxy 처리
  * 물리적으로 분리하여 보안 강화

#### 참고링크

* [https://mangkyu.tistory.com/14](https://mangkyu.tistory.com/14)
* [https://velog.io/@jakeseo\_me/자바-서블릿에-대해-알아보자.-근데-톰캣과-스프링을-살짝-곁들인](https://velog.io/@jakeseo\_me/%EC%9E%90%EB%B0%94-%EC%84%9C%EB%B8%94%EB%A6%BF%EC%97%90-%EB%8C%80%ED%95%B4-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90.-%EA%B7%BC%EB%8D%B0-%ED%86%B0%EC%BA%A3%EA%B3%BC-%EC%8A%A4%ED%94%84%EB%A7%81%EC%9D%84-%EC%82%B4%EC%A7%9D-%EA%B3%81%EB%93%A4%EC%9D%B8)
* [https://tecoble.techcourse.co.kr/post/2021-05-23-servlet-servletcontainer/](https://tecoble.techcourse.co.kr/post/2021-05-23-servlet-servletcontainer/)
* [https://gmlwjd9405.github.io/2018/10/27/webserver-vs-was.html](https://gmlwjd9405.github.io/2018/10/27/webserver-vs-was.html)
