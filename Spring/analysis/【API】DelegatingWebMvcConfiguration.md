>API官方地址：http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/DelegatingWebMvcConfiguration.html


org.springframework.web.servlet.config.annotation
**Class DelegatingWebMvcConfiguration**

```java
java.lang.Object
org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport
    org.springframework.web.servlet.config.annotation.DelegatingWebMvcConfiguration 

All Implemented Interfaces:
Aware, ApplicationContextAware, ServletContextAware 
```
---
```java
@Configuration
public class DelegatingWebMvcConfiguration
    extends WebMvcConfigurationSupport
```
`WebMvcConfigurationSupport`的子类，用于检测和委派所有类型为[WebMvcConfigurer](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/WebMvcConfigurer.html)的bean，
从而允许他们自定义由`WebMvcConfigurationSupport`提供的配置。这是[@EnableWebMvc](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/EnableWebMvc.html)实际导入的类。