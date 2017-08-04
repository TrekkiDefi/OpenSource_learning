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