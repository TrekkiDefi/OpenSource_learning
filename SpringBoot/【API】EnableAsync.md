>API官方地址：<http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/scheduling/annotation/EnableAsync.html>

org.springframework.scheduling.annotation **Annotation Type EnableAsync**

```java
@Target(value=TYPE)
@Retention(value=RUNTIME)
@Documented
@Import(value=AsyncConfigurationSelector.class)
public @interface EnableAsync
```

启用Spring的异步方法执行功能，类似于Spring的`<task:*>`XML命名空间中的功能。
要与`@Configuration`类一起使用如下，为整个Spring应用程序上下文启用注释驱动的异步处理：

```java
@Configuration
@EnableAsync
public class AppConfig {

}
```

`MyAsyncBean`是一个用户定义的类型，
其中包含一个或多个使用Spring的`@Async`注解，EJB 3.1 `@javax.ejb.Asynchronous`注解
或通过`annotation()`属性指定的任何自定义注解注释的方法。 

```java
@Configuration
public class AnotherAppConfig {

 @Bean
 public MyAsyncBean asyncBean() {
     return new MyAsyncBean();
 }
}
```

`mode()`属性控制通知如何应用; 

如果模式是[AdviceMode.PROXY](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/context/annotation/AdviceMode.html#PROXY)（默认），则其他属性控制代理的行为。

请注意，如果`mode()`设置为[AdviceMode.ASPECTJ](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/context/annotation/AdviceMode.html#ASPECTJ)，则将忽略`proxyTargetClass()`属性的值。 
还要注意，在这种情况下，spring-aspects模块JAR必须存在于类路径上。

默认情况下，Spring将查找一个关联的线程池定义：上下文中是唯一的[TaskExecutor](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/core/task/TaskExecutor.html)bean，

否则命名为"taskExecutor"的[Executor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html?is-external=true)bean。 

如果两者都不可解析，则[SimpleAsyncTaskExecutor](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/core/task/SimpleAsyncTaskExecutor.html)将用于处理异步方法调用。

此外，具有void返回类型的注解方法不能将任何异常传回给调用者。 默认情况下，仅日志记录此类未捕获的异常。

要自定义所有这一切，实现[AsyncConfigurer](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/scheduling/annotation/AsyncConfigurer.html)并提供：

- 你自定义的[Executor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html?is-external=true)通过`getAsyncExecutor()`方法
- 你自定义的`AsyncUncaughtExceptionHandler`，通过`getAsyncUncaughtExceptionHandler()`方法。

```java
@Configuration
@EnableAsync
public class AppConfig implements AsyncConfigurer {

 @Override
 public Executor getAsyncExecutor() {
     ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
     executor.setCorePoolSize(7);
     executor.setMaxPoolSize(42);
     executor.setQueueCapacity(11);
     executor.setThreadNamePrefix("MyExecutor-");
     executor.initialize();
     return executor;
 }

 @Override
 public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
     return MyAsyncUncaughtExceptionHandler();
 }
}
```

如果只需要自定义其中一项，可以返回null以使用默认设置。
也可以考虑扩展[AsyncConfigurerSupport](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/scheduling/annotation/AsyncConfigurerSupport.html)。

注意：在上面的例子中，`ThreadPoolTaskExecutor`不是一个完全管理的Spring bean。
如果您需要完全管理的bean，请将`@Bean`注解添加到`getAsyncExecutor()`方法。在这种情况下，不再需要手动调用`executor.initialize()`方法，因为这将在bean初始化时自动调用。

作为参考，上述示例可以与以下Spring XML配置进行比较：

```xml
<beans>

    <task:annotation-driven executor="myExecutor" exception-handler="exceptionHandler"/>
    
    <task:executor id="myExecutor" pool-size="7-42" queue-capacity="11"/>
    
    <bean id="asyncBean" class="com.foo.MyAsyncBean"/>
    
    <bean id="exceptionHandler" class="com.foo.MyAsyncUncaughtExceptionHandler"/>

</beans>
```

上述基于XML和基于JavaConfig的示例是等效的，除了设置Executor的线程名称前缀; 这是因为`<task:executor>`元素不会暴露这样的属性。
这演示了基于JavaConfig的方法如何通过直接访问实际组件来实现最大的可配置性。