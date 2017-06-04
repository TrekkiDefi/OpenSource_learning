>原文：<http://blog.csdn.net/xiaoyu411502/article/details/52770723>

`Spring Boot`是一个偏执的开源框架，它可用于创建可执行的Spring应用程序，采用了**习惯优于配置**的方法。 

>**约定优于配置**（convention over configuration），也称作**按约定编程**，是**一种软件设计范式**，旨在减少软件开发人员需做决定的数量，获得简单的好处，而又不失灵活性。
>
>**本质**是说，开发人员仅需规定应用中不符约定的部分。例如，如果模型中有个名为Sale的类，那么数据库中对应的表就会默认命名为sales。
>只有在偏离这一约定时，例如将该表命名为”products_sold”，才需写有关这个名字的配置。

此框架的神奇之处在于`@EnableAutoConfiguration`注释，此**注释自动载入应用程序所需的所有Bean**——这依赖于Spring Boot在类路径中的查找。

## 一、@Enable*注释
`@Enable*`注释并不是新发明的注释，早在Spring 3框架就引入了这些注释，用这些注释替代XML配置文件。 

很多Spring开发者都知道@EnableTransactionManagement注释，它能够声明事务管理；@EnableWebMvc注释，它能启用Spring MVC；以及@EnableScheduling注释，它可以初始化一个调度器。 
这些注释事实上都是简单的配置，通过@Import注释导入。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({ EnableAutoConfigurationImportSelector.class,
        AutoConfigurationPackages.Registrar.class })
public @interface EnableAutoConfiguration {

    /**
     * Exclude specific auto-configuration classes such that they will never be applied.
     */
    Class<?>[] exclude() default {};

}
```
`EnableAutoConfigurationImportSelector`类使用了**Spring Core包**的`SpringFactoriesLoader`类的`loadFactoryNames()`方法。 

`SpringFactoriesLoader`会查询具有`META-INF/spring.factories`文件的JAR包。 
当找到`spring.factories`文件后，`SpringFactoriesLoader`将查询配置文件命名的属性。

在例子中，是`org.springframework.boot.autoconfigure.EnableAutoConfiguration`。 

让我们来看看spring-boot-autoconfigure JAR文件，它真的包含了一个spring.factories文件，内容如下：
```properties
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.logging.AutoConfigurationReportLoggingInitializer

# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.PropertyPlaceholderAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.data.JpaRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.MongoRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.redis.RedisAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.JmsTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.jmx.JmxAutoConfiguration,\
org.springframework.boot.autoconfigure.mobile.DeviceResolverAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.MongoTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration,\
org.springframework.boot.autoconfigure.reactor.ReactorAutoConfiguration,\
org.springframework.boot.autoconfigure.security.SecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.FallbackWebSecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.thymeleaf.ThymeleafAutoConfiguration,\
org.springframework.boot.autoconfigure.web.EmbeddedServletContainerAutoConfiguration,\
org.springframework.boot.autoconfigure.web.DispatcherServletAutoConfiguration,\
org.springframework.boot.autoconfigure.web.ServerPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.web.MultipartAutoConfiguration,\
org.springframework.boot.autoconfigure.web.HttpMessageConvertersAutoConfiguration,\
org.springframework.boot.autoconfigure.web.WebMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.WebSocketAutoConfiguration
```
在这个文件中，可以看到**一系列Spring Boot自动配置的列表**。

下面我们来看这些配置的细节，以`MongoAutoConfiguration`为例：
```java
@Configuration
@ConditionalOnClass(Mongo.class)
@EnableConfigurationProperties(MongoProperties.class)
public class MongoAutoConfiguration {
    private Mongo mongo;

    @Autowired
    private MongoProperties properties;

    
    @PreDestroy
    public void close() throws UnknownHostException {
        if (this.mongo != null) {
            this.mongo.close();
        }
    }
    
    @Bean
    @ConditionalOnMissingBean
    public Mongo mongo() throws UnknownHostException {
        this.mongo = this.properties.createMongoClient();
        return this.mongo;
    }

}
```
这个类进行了简单的Spring配置，声明了MongoDB所需典型Bean。 

这个类跟其它很多类一样，重度依赖于Spring Boot注释： 
1. `@ConditionOnClass`当在类路径中存在这样的类时启用配置；
2. `@EnableConfigurationProperties`自动映射一个POJO到Spring Boot配置文件（默认是application.properties文件）的属性集；
3. `@ConditionalOnMissingBean`启用一个Bean定义，但必须是这个Bean之前未定义过才有效


还可以使用`@AutoConfigureBefore`注释、`@AutoConfigureAfter`注释来定义这些配置类的载入顺序。

## 二、属性映射
下面看MongoProperties类，它是一个Spring Boot属性映射的例子：
```java
@ConfigurationProperties(prefix = "spring.data.mongodb")
public class MongoProperties {

    private String host;
    private int port = DBPort.PORT;
    private String uri = "mongodb://localhost/test";
    private String database;

    // ... getters/ setters omitted
}
```
`@ConfigurationProperties`注释**将POJO关联到指定前缀的每一个属性**。例如，`spring.data.mongodb.port`属性将映射到这个类的端口属性。 

强烈建议Spring Boot开发者使用这种方式来删除与配置属性相关的瓶颈代码。

## 三、@Conditional注释
Spring Boot的强大之处在于使用了Spring 4框架的新特性：`@Conditional`注释，此**注释使得只有在特定条件满足时才启用一些配置**。 
在Spring Boot的`org.springframework.boot.autoconfigure.condition`包中说明了使用`@Conditional`注释能给我们带来什么，下面对这些注释做一个概述：


- @ConditionalOnBean
- @ConditionalOnClass
- @ConditionalOnExpression
- @ConditionalOnMissingBean
- @ConditionalOnMissingClass
- @ConditionalOnNotWebApplication
- @ConditionalOnResource
- @ConditionalOnWebApplication

以`@ConditionalOnExpression`注释为例，它允许在Spring的EL表达式中写一个条件。
```java
@Conditional(OnExpressionCondition.class)
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.TYPE, ElementType.METHOD })
public @interface ConditionalOnExpression {

    /**
     * The SpEL expression to evaluate. Expression should return {@code true} if the
     * condition passes or {@code false} if it fails.
     */
    String value() default "true";

}
```
在这个类中，我们想利用`@Conditional`注释，条件在`OnExpressionCondition`类中定义：
```java
public class OnExpressionCondition extends SpringBootCondition {

    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // ...
        // we first get a handle on the EL context via the ConditionContext

        boolean result = (Boolean) resolver.evaluate(expression, expressionContext);

        // ...
        // here we create a message the user will see when debugging

        return new ConditionOutcome(result, message.toString());
    }
}
```
在最后，@Conditional通过简单的布尔表达式（即ConditionOutcome方法）来决定。

## 四、应用程序上下文初始化器
`spring.factories`还**提供了第二种可能性**，即**定义应用程序的初始化**。这**使得我们可以在应用程序载入前操纵Spring的应用程序上下文ApplicationContext**。 

特别是，可以在上下文创建监听器，使用`ConfigurableApplicationContext`类的`addApplicationListener()`方法。 

`AutoConfigurationReportLoggingInitializer`监听到系统事件时，比如上下文刷新或应用程序启动故障之类的事件，Spring Boot可以执行一些工作。这有助于我们以调试模式启动应用程序时创建自动配置的报告。 

要以调试模式启动应用程序，可以使用`-Ddebug`标识，或者在`application.properties`文件这添加属性`debug=true`。

## 五、调试Spring Boot自动配置
Spring Boot的[官方文档](http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#howto-troubleshoot-auto-configuration)有助于理解自动配置期间发生了什么。

当以调试模式运行时，Spring Boot会产生一个报告，如下：
```text
Positive matches:
-----------------

MessageSourceAutoConfiguration
  - @ConditionalOnMissingBean (types: org.springframework.context.MessageSource; SearchStrategy: all) found no beans (OnBeanCondition)

JmxAutoConfiguration
  - @ConditionalOnClass classes found: org.springframework.jmx.export.MBeanExporter (OnClassCondition)
  - SpEL expression on org.springframework.boot.autoconfigure.jmx.JmxAutoConfiguration: ${spring.jmx.enabled:true} (OnExpressionCondition)
  - @ConditionalOnMissingBean (types: org.springframework.jmx.export.MBeanExporter; SearchStrategy: all) found no beans (OnBeanCondition)

DispatcherServletAutoConfiguration
  - found web application StandardServletEnvironment (OnWebApplicationCondition)
  - @ConditionalOnClass classes found: org.springframework.web.servlet.DispatcherServlet (OnClassCondition)


Negative matches:
-----------------

DataSourceAutoConfiguration
  - required @ConditionalOnClass classes not found: org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType (OnClassCondition)

DataSourceTransactionManagerAutoConfiguration
  - required @ConditionalOnClass classes not found: org.springframework.jdbc.core.JdbcTemplate,org.springframework.transaction.PlatformTransactionManager (OnClassCondition)

MongoAutoConfiguration
  - required @ConditionalOnClass classes not found: com.mongodb.Mongo (OnClassCondition)

FallbackWebSecurityAutoConfiguration
  - SpEL expression on org.springframework.boot.autoconfigure.security.FallbackWebSecurityAutoConfiguration: !${security.basic.enabled:true} (OnExpressionCondition)

SecurityAutoConfiguration
  - required @ConditionalOnClass classes not found: org.springframework.security.authentication.AuthenticationManager (OnClassCondition)

EmbeddedServletContainerAutoConfiguration.EmbeddedJetty
  - required @ConditionalOnClass classes not found: org.eclipse.jetty.server.Server,org.eclipse.jetty.util.Loader (OnClassCondition)

WebMvcAutoConfiguration.WebMvcAutoConfigurationAdapter#localeResolver
  - @ConditionalOnMissingBean (types: org.springframework.web.servlet.LocaleResolver; SearchStrategy: all) found no beans (OnBeanCondition)
  - SpEL expression: '${spring.mvc.locale:}' != '' (OnExpressionCondition)

WebSocketAutoConfiguration
  - required @ConditionalOnClass classes not found: org.springframework.web.socket.WebSocketHandler,org.apache.tomcat.websocket.server.WsSci (OnClassCondition)
```
对于每个自动配置，可以看到它启动或失败的原因。

## 六、结论
Spring Boot很好地利用了Spring 4框架的功能，可以创建一个自动配置的可执行JAR。 
不要忘记，可以使用自己的配置来替代自动配置，见：http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#using-boot-replacing-auto-configuration 