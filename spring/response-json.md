# 스프링, Response에서 Json까지

#### Response 직렬화 과정

1. `DispatcherServlet`
   1. `getHandlerAdapter()` 를 통해 적절한 HandlerAdaters 선택
   2. `handlerAdapter.handle()` 를 수행
2. `RequestMappingHandlerAdapter`
3. `HandlerMethodReturnValueHandlerComposite`
   1. `selectHandler()` 에서 리턴 타입에 맞는 `HandlerMethodReturnValueHandler` 을 선택
   2. `RequestResponseBodyMethodProcessor.supportsReturnType()` 에서 함수에서 클래스나 메서드에 ResponseBody 어노테이션이 있는지 체크 → true를 반환
   3. `RequestResponseBodyMethodProcessor` 를 Handler로 선택
4. `RequestResponseBodyMethodProcessor.handleReturnValue()`
   1. mavContainer.setRequestHandled(**true**)
   2. `writeWithMessageConverters()` 호출
5. `AbstractMessageConverterMethodProcessor`
   1. writeWithMessageConverters
      1. `getProducibleMediaTypes()`을 통해 가능한 응답타입 체크
         1. `getAcceptableMediaTypes()`를 통해 헤더의 accept 체크
      2. determineCompatibleMediaTypes을 통해 selectedMediaType과 producibleTypes을 비교해서 가능한 Type 처리
6. `AbstractJackson2HttpMessageConverter`
   1. `writeInternal()`을 통해 JSON 변환

#### 스프링 MVC 호출과정

```java
@RestController
public class TestController {
    @GetMapping("/hello")
    public String hello() {
        return "hello";
    }
```

TestController에서 “hello”가 어떻게 응답으로 내려가는지 그 과정을 살펴보자.

#### DispatcherServlet

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	// 윗부분은 생략
		HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

		// Process last-modified header, if supported by the handler.
		String method = request.getMethod();
		boolean isGet = HttpMethod.GET.matches(method);
		if (isGet || HttpMethod.HEAD.matches(method)) {
			long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
			if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
				return;
			}
		}

		if (!mappedHandler.applyPreHandle(processedRequest, response)) {
			return;
		}

		// Actually invoke the handler.
		mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
	// 이하 생략
}

protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		if (this.handlerAdapters != null) {
			for (HandlerAdapter adapter : this.handlerAdapters) {
				if (adapter.supports(handler)) {
					return adapter;
				}
			}
		}
		throw new ServletException("No adapter for handler [" + handler +
				"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
	}
```

* getHandlerAdapter를 통해 적절한 HandlerAdapter를 가져오고 ha.handle()을 통해 요청을 수행하는 부분이 핵심이다.
* `RequestMappingHandlerAdapter` 를 통해 이후 작업이 진행된다

#### RequestMappingHandlerAdapter

```java
@Override
protected ModelAndView handleInternal(HttpServletRequest request,
		HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
		// 윗부분 생략
		
		ModelAndView mav;
		mav = invokeHandlerMethod(request, response, handlerMethod);
		
		// 이하 생략
		}
```

* `handleInternal`
  * `invokeHandlerMethod` 를 통해 핸들러 메서드를 실행하고 그 결과를 `ModelAndView` 로 전달받는다.
* `invokeHandlerMethod` 에서는 `invocableMethod` 의 `returnValueHandlers`를 설정한다.

#### HandlerMethodReturnValueHandlerComposite

```java
@Override
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
		ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

	HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
	if (handler == null) {
		throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
	}
	handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
}

@Nullable
private HandlerMethodReturnValueHandler selectHandler(@Nullable Object value, MethodParameter returnType) {
	boolean isAsyncValue = isAsyncReturnValue(value, returnType);
	for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
		if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
			continue;
		}
		if (handler.supportsReturnType(returnType)) {
			return handler;
		}
	}
	return null;
}
```

* `selectHandler(returnValue, returnType)` 를 통해 적절한`HandlerMethodReturnValueHandler` 를 찾는다.
* TestController에는 `@RestController` 어노테이션이 있으며, 해당 어노테이션에는 `@ResponseBody`가 포함되어 있다.
* 따라서 `this.returnValueHandlers` 중에서 `RequestResponseBodyMethodProcessor.supportsReturnType()` 을 호출했을 때, true를 반환하게 되어 최종적으로 `HandlerMethodReturnValueHandler`를 상속받는 `RequestResponseBodyMethodProcessor` 가 Handler로 선택된다.

```java
// RequestResponseBodyMethodProcessor 클래스
@Override
public boolean supportsReturnType(MethodParameter returnType) {
	return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class) ||
			returnType.hasMethodAnnotation(ResponseBody.class));
}
```

`supportsReturnType` 메서드 로직을 보면 ResponseBody 어노테이션이 클래스나 메서드에 있는지 확인하는 것을 알 수 있다.

#### RequestResponseBodyMethodProcessor

```java
@Override
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
		ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
		throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

	mavContainer.setRequestHandled(true);
	ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
	ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

	if (returnValue instanceof ProblemDetail detail) {
		outputMessage.setStatusCode(HttpStatusCode.valueOf(detail.getStatus()));
		if (detail.getInstance() == null) {
			URI path = URI.create(inputMessage.getServletRequest().getRequestURI());
			detail.setInstance(path);
		}
	}

	// Try even with null return value. ResponseBodyAdvice could get involved.
	writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
}

protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
			ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
	
	// 윗부분 생략 
	List<MediaType> acceptableTypes;
	try {
		acceptableTypes = getAcceptableMediaTypes(request);
	}
	
	// 중략
	
	List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);
	if (body != null && producibleTypes.isEmpty()) {
		throw new HttpMessageNotWritableException(
				"No converter found for return value of type: " + valueType);
	}

	List<MediaType> compatibleMediaTypes = new ArrayList<>();
	determineCompatibleMediaTypes(acceptableTypes, producibleTypes, compatibleMediaTypes);		
	
	// 중략 
	
	if (selectedMediaType != null) {
		selectedMediaType = selectedMediaType.removeQualityValue();
		for (HttpMessageConverter<?> converter : this.messageConverters) {
			GenericHttpMessageConverter genericConverter =
					(converter instanceof GenericHttpMessageConverter ghmc ? ghmc : null);
			if (genericConverter != null ?
					((GenericHttpMessageConverter) converter).canWrite(targetType, valueType, selectedMediaType) :
					converter.canWrite(valueType, selectedMediaType)) {
				body = getAdvice().beforeBodyWrite(body, returnType, selectedMediaType,
						(Class<? extends HttpMessageConverter<?>>) converter.getClass(),
						inputMessage, outputMessage);
				if (body != null) {
					Object theBody = body;
					LogFormatUtils.traceDebug(logger, traceOn ->
							"Writing [" + LogFormatUtils.formatValue(theBody, !traceOn) + "]");
					addContentDispositionHeader(inputMessage, outputMessage);
					if (genericConverter != null) {
						genericConverter.write(body, targetType, selectedMediaType, outputMessage);
					}
					else {
						((HttpMessageConverter) converter).write(body, selectedMediaType, outputMessage);
					}
				}
				else {
					if (logger.isDebugEnabled()) {
						logger.debug("Nothing to write: null body");
					}
				}
				return;
			}
		}
	}

	// 이하 생략			
}			
```

* `handleReturnValue` 메서드에서 mavContainer.setRequestHandled(true)를 설정한다.
  * `ModelAndViewContainer`에서 추가적인 뷰 렌더링이 필요하지 않다는 것을 표시한다. 이를 통해 뷰 리졸버 검색과 뷰 렌더링 과정을 건너뛸 수 있다.
  * `writeWithMessageConverters` 메서드를 통해 메시지컨버터가 write하도록 한다.
* `writeWithMessageConverters`
  * getAcceptableMediaTypes을 통해 클라이언트가 요청한 미디어 타입을 확인한다.
    * header의 accept 타입을 체크한다.
  * `getProducibleMediaTypes()` 에서는 컨트롤러 메서드가 생성 가능한 미디어 타입을 확인한다.
  * `determineCompatibleMediaTypes()` 메서드에서는 acceptableTypes와 producibleTypes를 비교해서 compatibleMediaTypes를 결정한다.
  * compatibleMediaTypes가 정해지고 나면 그에 맞는 `HttpMessageConverter` 를 정한다.
    * `@RestController`를 사용하고 선택된 미디어 타입 `application/json`이면 기본적으로 MappingJackson2HttpMessageConverter가 선택된다.

#### AbstractJackson2HttpMessageConverter

```java
@Override
protected void writeInternal(Object object, @Nullable Type type, HttpOutputMessage outputMessage) {
// 윗부분은 생략

objectWriter.writeValue(generator, value);
// 이하 생략
}
```

* 최종적으로 ObjectWriter.writeValue()를 통해 객체를 JSON으로 직렬화하고 `JsonGenerator` 에 작성한다.

#### 참고문서

* [https://bepoz-study-diary.tistory.com/374](https://bepoz-study-diary.tistory.com/374)
* [https://happy-coding-day.tistory.com/entry/SpringMVC-에서-말하는-MessageConverter-코드로-이해하기](https://happy-coding-day.tistory.com/entry/SpringMVC-%EC%97%90%EC%84%9C-%EB%A7%90%ED%95%98%EB%8A%94-MessageConverter-%EC%BD%94%EB%93%9C%EB%A1%9C-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0)
* [https://blog.naver.com/gngh0101/221516352327](https://blog.naver.com/gngh0101/221516352327)
* [https://rlaehddnd0422.tistory.com/61](https://rlaehddnd0422.tistory.com/61)
