>API官方地址：<http://docs.spring.io/autorepo/docs/spring-boot/1.2.2.RELEASE/api/org/springframework/boot/context/embedded/ServletContextInitializer.html>

org.springframework.boot.context.embedded **Interface ServletContextInitializer**

```text
All Known Implementing Classes:
        FilterRegistrationBean, InitParameterConfiguringServletContextInitializer, RegistrationBean, ServletListenerRegistrationBean, ServletRegistrationBean
```

```java
public interface ServletContextInitializer
```

用于以编程方式配置Servlet 3.0+上下文的接口。

与`WebApplicationInitializer`不同，`SpringServletContainerInitializer`将不会检测到实现此接口（并且不实现`WebApplicationInitializer`）的类，
因此不会被Servlet容器自动引导。

此接口主要用于允许`ServletContextInitializer`由Spring管理，而不是Servlet容器。