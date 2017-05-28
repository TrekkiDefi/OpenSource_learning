>官方API地址：http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/EnableWebMvc.html

org.springframework.web.servlet.config.annotation
**Annotation Type EnableWebMvc**

```java
@Retention(value=RUNTIME)
@Target(value=TYPE)
@Documented
@Import(value=DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc
```
将此注释添加到`@Configuration`类会从[WebMvcConfigurationSupport](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/WebMvcConfigurationSupport.html)导入Spring MVC配置，例如：
```java
@Configuration
@EnableWebMvc
@ComponentScan(basePackageClasses = { MyConfiguration.class })
public class MyWebConfiguration {

}
```

要自定义导入的配置，实现[WebMvcConfigurer](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/WebMvcConfigurer.html)接口或更好的方式是扩展空方法基类[WebMvcConfigurerAdapter](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/WebMvcConfigurerAdapter.html)并覆盖单个方法，例如：

```java
@Configuration
@EnableWebMvc
@ComponentScan(basePackageClasses = { MyConfiguration.class })
public class MyConfiguration extends WebMvcConfigurerAdapter {

       @Override
       public void addFormatters(FormatterRegistry formatterRegistry) {
     formatterRegistry.addConverter(new MyConverter());
       }

       @Override
       public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
     converters.add(new MyHttpMessageConverter());
       }

 // More overridden methods ...
}
```

如果[WebMvcConfigurer](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/WebMvcConfigurer.html)未暴露一些需要配置的高级设置，
请考虑删除`@EnableWebMvc`注释，并直接从[WebMvcConfigurationSupport](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/WebMvcConfigurationSupport.html)或[DelegatingWebMvcConfiguration](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/DelegatingWebMvcConfiguration.html)扩展，例如：

```java
@Configuration
@ComponentScan(basePackageClasses = { MyConfiguration.class })
public class MyConfiguration extends WebMvcConfigurationSupport {

       @Override
       public void addFormatters(FormatterRegistry formatterRegistry) {
     formatterRegistry.addConverter(new MyConverter());
       }

       @Bean
       public RequestMappingHandlerAdapter requestMappingHandlerAdapter() {
     // Create or delegate to "super" to create and
     // customize properties of RequestMappingHandlerAdapter
       }
}
```

```java
Since:
    3.1
Author:
    Dave Syer, Rossen Stoyanchev
See Also:
    WebMvcConfigurer, WebMvcConfigurerAdapter, WebMvcConfigurationSupport, DelegatingWebMvcConfiguration 
```