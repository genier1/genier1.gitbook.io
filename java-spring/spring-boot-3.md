# Spring boot 3 구동과정

Spring Boot 3.1 기준 스프링 코드는 다음과 같다.

```java
public ConfigurableApplicationContext run(String... args) {
        if (this.registerShutdownHook) {
            shutdownHook.enableShutdowHookAddition();
        }

        long startTime = System.nanoTime();
        // 1. BootstrapContext 생성 
        DefaultBootstrapContext bootstrapContext = this.createBootstrapContext();
        ConfigurableApplicationContext context = null;
        // 2. Java AWT Headless Property 설정
        this.configureHeadlessProperty();
        // 3. SpringApplicationRunListener 로드 및 starting
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        listeners.starting(bootstrapContext, this.mainApplicationClass);

        Throwable ex;
        try {
		        // 4. Arguments 래핑 
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
	          // 5. Environment 준비
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, bootstrapContext, applicationArguments);
            // 6. 배너 출력 
            Banner printedBanner = this.printBanner(environment);
            // 7. applicationContext 생성
            context = this.createApplicationContext();
            context.setApplicationStartup(this.applicationStartup);
            // 8. Context 준비
            this.prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
            // 9. Context Refresh
            this.refreshContext(context);
            // 10. Context Refresh 후처리
            this.afterRefresh(context, applicationArguments);
            Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), timeTakenToStartup);
            }
            // 11. SpringApplicationRunListener started 처리
            listeners.started(context, timeTakenToStartup);
            // 12. Runners 실행
            this.callRunners(context, applicationArguments);
        }
        
        ... 이하 생략 
```

1. BootstrapContext 생성
   * BootStrapContext란 애플리케이션 컨텍스트가 준비될 때 까지 환경 변수들을 관리하는 스프링의 Environment 객체를 후처리하기 위한 임시 컨텍스트이다.
2. Java AWT Headless Property 설정
   * Java AWT Headless 모드는 모니터나 마우스, 키보드 등의 디스플레이 장치가 없는 서버 환경에서 UI 클래스를 사용할 수 있도록 하는 옵션이다.
3. SpringApplicationRunListener 로드 및 starting
   * SpringApplicationRunListener는 애플리케이션이 시작되고 실행되는 과정에서 발생하는 주요 이벤트를 수신하고 처리하는 역할을 한다.
   * starting()을 통해 각 Listener에게 이벤트를 전송한다.
   * starting()외에도 environmentPrepared(), contextPrepared(), contextLoaded(), started(), running(), failed() 등 다양한 이벤트를 처리할 수 있다.
4. Arguments 래핑
   * ApplicationArguments SpringApplication의 run 메서드 실행 시 제공한 인자(args)에 접근하고 활용할 수 있는 객체이다.
5. Environment 준비
   * ConfigurableEnvironment는 Spring boot 애플리케이션의 environment 값과 관련된 클래스다.
   * 해당 클래스를 통해 Property source를 지정하고 우선순위를 변경할 수 있다.
6.  배너 출력

    * 디폴트로 아래의 배너를 출력한다.

    ```java
      .   ____          _            __ _ _
     /\\\\ / ___'_ __ _ _(_)_ __  __ _ \\ \\ \\ \\
    ( ( )\\___ | '_ | '_| | '_ \\/ _` | \\ \\ \\ \\
     \\\\/  ___)| |_)| | | | | || (_| |  ) ) ) )
      '  |____| .__|_| |_|_| |_\\__, | / / / /
     =========|_|==============|___/=/_/_/_/
     :: Spring Boot ::                (v3.1.4)
    ```

    * [배너 커스터마이징](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#features.spring-application.banner)도 가능하다.
7. 애플리케이션 컨텍스트 생성
   * `WebApplicationType.*deduceFromClasspath*()` 을 통해 결정된 webApplicationType을 기반으로 ApplicationContextFactory의 구현체를 선택하여 ApplicationContext를 생성한다.
   * 기본값은 `DefaultApplicationContextFactory` 을 사용하게 된다.
8.  Context 준비

    ```java
    private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context, ConfigurableEnvironment environment, SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
            context.setEnvironment(environment);
            this.postProcessApplicationContext(context);
            this.addAotGeneratedInitializerIfNecessary(this.initializers);
            this.applyInitializers(context);
            listeners.contextPrepared(context);
            bootstrapContext.close(context);
            if (this.logStartupInfo) {
                this.logStartupInfo(context.getParent() == null);
                this.logStartupProfileInfo(context);
            }

            ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
            beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
            if (printedBanner != null) {
                beanFactory.registerSingleton("springBootBanner", printedBanner);
            }

            if (beanFactory instanceof AbstractAutowireCapableBeanFactory autowireCapableBeanFactory) {
                autowireCapableBeanFactory.setAllowCircularReferences(this.allowCircularReferences);
                if (beanFactory instanceof DefaultListableBeanFactory listableBeanFactory) {
                    listableBeanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
                }
            }

            if (this.lazyInitialization) {
                context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
            }

            context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context));
            if (!AotDetector.useGeneratedArtifacts()) {
                Set<Object> sources = this.getAllSources();
                Assert.notEmpty(sources, "Sources must not be empty");
                this.load(context, sources.toArray(new Object[0]));
            }

            listeners.contextLoaded(context);
        }
    ```

    * `context.setEnvironment(environment)`를 호출해서ApplicationContext에 환경(Environment) 객체를 설정한다.
    * `this.postProcessApplicationContext(context)` 를 호출해서 ApplicationContext에 대한 후처리 작업을 수행한다.
      * `beanNameGenerator(빈 이름 지정 클래스), resourceLoader(리소스를 불러오는 클래스), conversionService(프로퍼티의 타입 변환)`를 ApplicationContext에 설정한다.
    * `this.applyInitializers(context)`를 호출해서 등록된 `ApplicationContextInitializer`들을 적용한다.
      * ApplicationContextInitializer는 Spring의 ConfigurableApplicationContext가 초기화되는 과정에서, refresh 되기 전 callback 메서드를 실행하기 위한 인터페이스이다.
    * `listeners.contextPrepared(context)`를 호출해서ApplicationContext가 준비되었음을 리스너에게 알린다.
    * `bootstrapContext.close(context)` 를 호출해서 부트스트랩 컨텍스트를 닫는다.
    * `ApplicationContext`의 `beanFactory`에 "springApplicationArguments" 빈을 등록하고, "springBootBanner" 빈도 등록한다.
    * `beanFactory`가 `AbstractAutowireCapableBeanFactory`의 인스턴스이면, `allowCircularReferences`와 `allowBeanDefinitionOverriding` 설정을 적용한다.
    * `lazyInitialization`이 true이면, `LazyInitializationBeanFactoryPostProcessor`를 등록한다.
    * `PropertySourceOrderingBeanFactoryPostProcessor` 를 등록한다.
      * PropertySource 객체의 순서를 조정하는 역할을 하는 클래스이다.
    * `this.load(context, sources.toArray(new Object[0]));`
      * ApplicationContext에 bean을 등록한다.
    * ApplicationContext에 bean을 등록하고, `listeners.contextLoaded(context)`를 호출하여 ApplicationContext가 로드되었음을 리스너에게 알린다.
9. Context Refresh
   * Spring boot의 configuration과 관련된 작업을 진행한다.
   * 별도의 글로 등록 예정
10. Context Refresh 후처리
    * ApplicationContext가 refresh 된 이후 호출되는 함수다. 해당 메서드를 Override 함으로써 추가적인 로직을 수행할 수 있다
11. SpringApplicationRunListener started 처리
    * 리스너에게 ApplicationContext가 refresh 된 후 started 된 상태임을 알린다.
12. Runners 실행
    * `ApplicationRunner`와 `CommandLineRunner` 인터페이스를 구현한 빈(bean)들을 실행하는 역할을 한다.
    * 어플리케이션이 실행된 이후에 초기화 작업을 필요로 하는 경우 Runner를 등록해 사용하면 된다.

#### 참고링크

* [https://lingi04.tistory.com/30](https://lingi04.tistory.com/30)
* [https://mangkyu.tistory.com/212](https://mangkyu.tistory.com/212)
* [https://code-run.tistory.com/5](https://code-run.tistory.com/5)
* [https://blog.naver.com/gngh0101/222195371984](https://blog.naver.com/gngh0101/222195371984)
