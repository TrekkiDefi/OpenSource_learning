>API官方地址：http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/WebMvcConfigurationSupport.html

org.springframework.web.servlet.config.annotation
**Class WebMvcConfigurationSupport**
```java
java.lang.Object
    org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport

All Implemented Interfaces:
    Aware, ApplicationContextAware, ServletContextAware


Direct Known Subclasses:
    DelegatingWebMvcConfiguration
```
---
```java
public class WebMvcConfigurationSupport
    extends Object
    implements ApplicationContextAware, ServletContextAware
```

这是提供`MVC Java config`背后配置的主要类。 通常**通过将`@EnableWebMvc`添加到应用程序`@Configuration`类来导入**。 

另一种更高级的方式是直接继承此类，并根据需要覆盖方法，记住将`@Configuration`添加到子类，将`@Bean`添加到覆盖的@Bean方法中。 

有关详细信息，请参阅[@EnableWebMvc](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/EnableWebMvc.html)的Javadoc。

### 此类注册以下[HandlerMappings](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/HandlerMapping.html)：

- [RequestMappingHandlerMapping](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/RequestMappingHandlerMapping.html)排序索引:0，用于将请求映射到带注释的控制器方法。 
- [HandlerMapping](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/HandlerMapping.html)排序索引:1，直接映射URL路径到视图名称。 
- [BeanNameUrlHandlerMapping](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/handler/BeanNameUrlHandlerMapping.html)排序索引:2，将URL路径映射到控制器bean名称。 
- [HandlerMapping](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/HandlerMapping.html)排序索引:Integer.MAX_VALUE-1，以提供静态资源请求。 
- [HandlerMapping](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/HandlerMapping.html)排序索引:Integer.MAX_VALUE，命令将请求转发到默认servlet。

### 注册这些[HandlerAdapters](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/HandlerAdapter.html)：

- [RequestMappingHandlerAdapter](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/RequestMappingHandlerAdapter.html)用于使用带注释的控制器方法处理请求。 
- [HttpRequestHandlerAdapter](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/mvc/HttpRequestHandlerAdapter.html)用于使用`HttpRequestHandlers`处理请求。 
- [SimpleControllerHandlerAdapter](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/mvc/SimpleControllerHandlerAdapter.html)用于使用`interface-based`控制器处理请求。

### 用这个异常解析器链注册一个[HandlerExceptionResolverComposite](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/handler/HandlerExceptionResolverComposite.html)：

- 用于通过[@ExceptionHandler](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/ExceptionHandler.html)方法处理异常的[ExceptionHandlerExceptionResolver](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/ExceptionHandlerExceptionResolver.html)。 
- 用[@ResponseStatus](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/ResponseStatus.html)注释的异常的[ResponseStatusExceptionResolver](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/mvc/annotation/ResponseStatusExceptionResolver.html)。 
- 用于解析已知的Spring异常类型的[DefaultHandlerExceptionResolver](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/mvc/support/DefaultHandlerExceptionResolver.html)

### 注册[AntPathMatcher](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/util/AntPathMatcher.html)和[UrlPathHelper](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/util/UrlPathHelper.html)以供以下功能使用： 
- [RequestMappingHandlerMapping](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/RequestMappingHandlerMapping.html)， 
- ViewController的[HandlerMapping](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/HandlerMapping.html) 
- 以及用于服务资源的[HandlerMapping](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/HandlerMapping.html)

请注意，这些bean可以使用[PathMatchConfigurer](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/PathMatchConfigurer.html)进行配置。

### 默认情况下，[RequestMappingHandlerAdapter](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/RequestMappingHandlerAdapter.html)和[ExceptionHandlerExceptionResolver](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/ExceptionHandlerExceptionResolver.html)都配置有以下默认实例： 
- [ContentNegotiationManager](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/accept/ContentNegotiationManager.html) 
- [DefaultFormattingConversionService](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/format/support/DefaultFormattingConversionService.html) 
- 如果在类路径上可用JSR-303实现，[OptionalValidatorFactoryBean](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/validation/beanvalidation/OptionalValidatorFactoryBean.html)
- 一系列的依赖类路径上可用的第三方库的[HttpMessageConverters](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/http/converter/HttpMessageConverter.html)。

```

Since:
    3.1
Author:
    Rossen Stoyanchev, Brian Clozel, Sebastien Deleuze
See Also:
    EnableWebMvc, WebMvcConfigurer, WebMvcConfigurerAdapter 
```