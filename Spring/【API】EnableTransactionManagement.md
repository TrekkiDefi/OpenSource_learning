org.springframework.transaction.annotation 

**Annotation Type EnableTransactionManagement**

```text
@Target(value=TYPE)
@Retention(value=RUNTIME)
@Documented
@Import(value=TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement
```

启用Spring注释驱动的事务管理功能，类似于Spring`<tx:*>`XML命名空间中的支持。 要在@Configuration类中使用如下：

```java
@Configuration
@EnableTransactionManagement
public class AppConfig {

 @Bean
 public FooRepository fooRepository() {
     // configure and return a class having @Transactional methods
     return new JdbcFooRepository(dataSource());
 }

 @Bean
 public DataSource dataSource() {
     // configure and return the necessary JDBC DataSource
 }

 @Bean
 public PlatformTransactionManager txManager() {
     return new DataSourceTransactionManager(dataSource());
 }
}
```

作为参考，上述示例可以与以下Spring XML配置进行比较：

```xml
<beans>
    <tx:annotation-driven/>
    
    <bean id="fooRepository" class="com.foo.JdbcFooRepository">
     <constructor-arg ref="dataSource"/>
    </bean>
    
    <bean id="dataSource" class="com.vendor.VendorDataSource"/>
    
    <bean id="transactionManager" class="org.sfwk...DataSourceTransactionManager">
     <constructor-arg ref="dataSource"/>
    </bean>
</beans>
```

在上述两种情况下，`@EnableTransactionManagement`和`<tx:annotation-driven />`都负责注册注解驱动事务管理的必要的Spring组件，
例如TransactionInterceptor和基于代理或基于AspectJ的advice，当JdbcFooRepository的`@Transactional`方法被调用时，将拦截器织入调用堆栈。

两个示例之间的细微差别在于`PlatformTransactionManager` bean的命名：在`@Bean`的情况下，名称为"txManager"（根据方法的名称）; 
在XML的情况下，名称是"transactionManager"。默认情况下，`<tx:annotation-driven/>`是来查找名为"transactionManager"的bean(根据名称查找)，

但是`@EnableTransactionManagement`更灵活; 它将根据类型查找容器中的PlatformTransactionManager bean。 
因此，名称可以是"txManager", "transactionManager", 或者 "tm"：这根本没关系。

对于那些希望在`@EnableTransactionManagement`和要使用的确切事务管理器bean之间建立更直接的关系的人，可以实现`TransactionManagementConfigurer`回调接口

- 注意以下的implements子句和`@Override-annotated`方法：

```java
@Configuration
@EnableTransactionManagement
public class AppConfig implements TransactionManagementConfigurer {

 @Bean
 public FooRepository fooRepository() {
     // configure and return a class having @Transactional methods
     return new JdbcFooRepository(dataSource());
 }

 @Bean
 public DataSource dataSource() {
     // configure and return the necessary JDBC DataSource
 }

 @Bean
 public PlatformTransactionManager txManager() {
     return new DataSourceTransactionManager(dataSource());
 }

 @Override
 public PlatformTransactionManager annotationDrivenTransactionManager() {
     return txManager();
 }
}
```

这种方法可能只是因为它更加明确，或者为了区分同一容器中存在的两个PlatformTransactionManager bean。
顾名思义，`annotationDrivenTransactionManager()`将用于处理`@Transactional`方法。

有关更多详细信息，请参阅[TransactionManagementConfigurer](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/transaction/annotation/TransactionManagementConfigurer.html) Javadoc。

`mode()`属性控制如何应用`advice`; 如果模式是`AdviceMode.PROXY`（默认），则其他属性控制代理的行为。

如果`mode()`设置为`AdviceMode.ASPECTJ`，则`proxyTargetClass() `属性已过时。 

还要注意，在这种情况下，spring-aspects模块JAR包必须存在于类路径上。