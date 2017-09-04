Spring通过其`PlatformTransactionManager`接口和实现对事务管理提供了丰富的支持。
Spring的事务支持为许多API的事务语义提供了一致的接口。广义上，事务可以分为两类：本地事务和全局事务。
本地事务是仅影响一个事务资源的事务。 大多数情况下，这些资源都有自己的事务API，即使事务的概念没有明确浮出水面。
通常，它会作为一个会话的概念浮出水面，一个`unit of work`，具有`demarcating APIs`，以便告诉资源缓存的工作何时应该提交。
全局事务跨越一个或多个事务资源，并将其全部纳入在单个事务中。

本地事务的一些常见示例在JMS和JDBC API中。
在JMS中，用户可以创建一个事务处理的会话，发送和接收消息，当消息的工作完成后，调用`Session.commit()`来告诉服务器它可以完成工作。
在数据库世界中，JDBC Connections默认自动提交查询。这对于一次性语句是很好的，但通常最好将一些相关的语句收集到一个批处理中，然后全部提交它们或者全部不提交。
在JDBC中，首先将Connection的`setAutoCommit()`方法设置为false，然后在批处理结束时显式调用`Connection.commit()`来执行此操作。
这两个API和其他许多API都提供了一个`transactional unit-of-work`的概念，它们可能会由客户端自行决定`committed, finalized, flushed`或以其他方式持久化。
API广泛不同，但概念是一样的。

全局事务完全不同。只要您希望多个资源参与到一个事务中，就应该使用它们。
可能会有这样的需求：也许你想发送一个JMS消息并写入数据库？ 或者，也许你想使用两种不同的JPA持久性上下文？ 
在全局事务设置中，第三方事务监控器在一个事务中纳入多个事务资源，准备提交
- 在此阶段，资源通常执行相当于`dry-run`提交
- 然后最终提交每个资源。

这些步骤是大多数全局事务实现的基础，被称为两阶段提交（2PC）。如果一个提交失败（就像网络中断），则事务监视器会请求每个资源撤消或回滚最后一个事务。

全局事务比常规的本地事务显得更复杂，因为根据定义，它们必须涉及第三方代理，其唯一功能是仲裁若干事务资源之间的事务状态。
事务监视器还必须知道如何与每个事务资源进行通信。因为这个代理还必须保持状态 
- 毕竟，不能信任个别的事务资源来知道其他资源正在做什么
- 它具有持久性要求和同步成本，以保持一致的事务日志，非常像数据库。

当发生灾难性事件并且事务监视器关闭时，它必须能够启动并重启运行中的事务，以确保所有资源处于一致状态。
为了与事务资源进行通信，事务监视器与事务资源必须使用通用协议通信。通常，该协议是一个称为XA的规范。

在Java企业开发中，将XA添加到应用程序的典型方法是使用JTA。Java Transaction API（JTA）是描述用户的标准全局事务监视器API的规范。

您可以使用JTA API以支持它的资源类型的标准方式处理全局事务 
- 通常只是JDBC和JMS。

Java EE应用程序服务器支持JTA开箱即用，还有JTA的第三方独立实现，可用于避免被困在Java EE应用程序服务器上。

要使用JTA，您需要做出非常具体的选择，将其用于事务代码中，因为该API与由JDBC公开的事务API非常不同，后者又具有与JMS完全不同的API。

看到针对JTA API编写的旧代码是非常常见的，即使只涉及到一个资源，因为如果稍后需要，至少不需要完全重写以利用XA。
这通常是一个令人遗憾但可以理解的决定。
以这种方式写的代码不幸也经常与容器绑在一起;它不会工作，除非JNDI和所有的服务器机器正在运行以引导事务监视器和JNDI服务器。

幸运的是，使用Spring可以优雅地避免这种复杂性。

## Spring's Transaction Support

任何解决这个令人遗憾的事情的第一部分是Spring `PlatformTransactionManager` API和Spring框架中关联的事务管理API。
Spring框架和周边项目提供了丰富的PlatformTransactionManager实现选择，支持JDBC，JMS，Gemfire，AMQP，Hibernate，JPA，JDO，iBatis等许多本地事务。

![](./images/TransactionClassHierarchy.png)

Spring框架还提供了`TransactionTemplate`，它与`PlatformTransactionManager`实现一起使用，可用于自动将一个`unit of work`包围在事务中，
这样您甚至不需要了解`PlatformTransactionManager` API本身，事务将被正确启动，执行，准备和提交，并且 - 如果在处理期间抛出异常，则事务将被回滚。

```java
@Inject
private PlatformTransactionManager txManager; 

TransactionTemplate template  = new TransactionTemplate(this.txManager); 
template.execute( new TransactionCallback<Object>(){ 
  public void doInTransaction(TransactionStatus status){ 
   // 这里执行的工作将被事务包裹并提交。事务将被回滚，如果`status.setRollbackOnly(true)`被调用或抛出异常
  } 
});
```

进一步说，Spring框架通过简单地支持它来为事务中的方法调用
提供了可靠的基于AOP的支持。通过此支持，您不再需要`TransactionTemplate` - 对于启用声明式事务管理注解的任何方法，事务管理会自动进行。

如果您使用Spring的XML配置，请使用以下命令：

```xml
<tx:annotation-driven transaction-manager = "platformTransactionManagerReference"/>
```

如果您使用Spring 3.1，那么您可以简单地注释Java配置类，如下所示：

```java
@EnableTransactionManagement
@Configuration 
public class MyConfiguration { 
  ... 
```

此注解将自动扫描您的bean，并查找类型为`PlatformTransactionManager`的bean。然后，在Java bean中，通过注释它们来简单地将方法声明为事务。

```java
@Transactional
public void work() { 
  // the transaction will be rolled back if the 
  // method throws an Exception, otherwise committed
}
```

## JTA

现在，如果你仍然需要使用JTA，至少可以选择你做的。有两种常见的情况：在重量级应用程序服务器中使用JTA，或使用独立的JTA实现。

Spring通过名为`JtaTransactionManager`的`PlatformTransactionManager`实现提供对基于JTA的全局事务实现的支持。

如果您在JavaEE应用程序服务器上使用它，它将自动从JNDI找到正确的`javax.transaction.UserTransaction`引用。
此外，它将尝试在9个不同的应用程序服务器中查找特定于容器的`javax.transaction.TransactionManager`引用，用于更高级的用例，如事务挂起。

在幕后，Spring可以加载不同的JtaTransactionManager子类，以便在可用时利用不同服务器中的特定功能，
例如：WebLogicJtaTransactionManager，WebSphereUowTransactionManager和OC4JJTTransactionManager。

因此，如果您在使用Java EE应用程序服务器，但希望使用Spring的JTA支持，则可以使用以下命名空间配置支持正确（并自动）产生`JtaTransactionManager`：

```xml
<tx:jta-transaction-manager  />
```

或者，您可以根据需要注册JtaTransactionManager bean实例，而不使用构造函数参数，如下所示：

```java
@Bean
public PlatformTransactionManager platformTransactionManager(){ 
    return new JtaTransactionManager();
}
```

无论哪种方式，JavaEE应用程序服务器的最终结果是，现在可以使用JTA以统一的方式管理事务，这要归功于Spring。
