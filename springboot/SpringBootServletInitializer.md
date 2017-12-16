>API官方地址：<http://docs.spring.io/autorepo/docs/spring-boot/1.2.2.RELEASE/api/org/springframework/boot/context/web/SpringBootServletInitializer.html>

org.springframework.boot.context.web **Class SpringBootServletInitializer**

```text
java.lang.Object
    org.springframework.boot.context.web.SpringBootServletInitializer 

All Implemented Interfaces:
    WebApplicationInitializer 
```

```java
public abstract class SpringBootServletInitializer
    extends Object
    implements WebApplicationInitializer
```

[WebApplicationInitializer](http://docs.spring.io/spring-framework/docs/4.1.5.RELEASE/javadoc-api/org/springframework/web/WebApplicationInitializer.html?is-external=true)
以传统的WAR部署运行
[SpringApplication](http://docs.spring.io/autorepo/docs/spring-boot/1.2.2.RELEASE/api/org/springframework/boot/SpringApplication.html)。

将应用程序上下文中的
[Servlet](https://docs.oracle.com/javaee/7/api/javax/servlet/Servlet.html?is-external=true)，
[Filter](https://docs.oracle.com/javaee/7/api/javax/servlet/Filter.html?is-external=true)和
[ServletContextInitializer](http://docs.spring.io/autorepo/docs/spring-boot/1.2.2.RELEASE/api/org/springframework/boot/context/embedded/ServletContextInitializer.html) 
bean绑定到servlet容器。

要配置应用程序，可以覆盖`configure(SpringApplicationBuilder)`方法（调用`SpringApplicationBuilder.sources(Object...)`）
或`make the initializer itself a @Configuration`。 

如果您将`SpringBootServletInitializer`与其他`WebApplicationInitializer`结合使用，
您可能还需要添加一个`@Ordered`注释来配置特定的启动顺序。

请注意，只有在构建war文件并进行部署时，才需要`WebApplicationInitializer`。
如果你运行一个嵌入式的容器，那么你根本不需要这个。