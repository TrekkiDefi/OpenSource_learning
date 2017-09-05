Spring通过其`PlatformTransactionManager`接口和实现对事务管理提供了丰富的支持。
Spring的事务支持为许多API的事务语义提供了一致的接口。广义上，事务可以分为两类：本地事务和全局事务。
本地事务是仅影响一个事务资源的事务。 大多数情况下，这些资源都有自己的事务API，即使事务的概念没有明确浮出水面。
通常，它会作为一个会话的概念浮出水面，一个`unit of work`，具有`demarcating APIs`，以便告诉资源缓存的工作何时应该提交。
全局事务跨越一个或多个事务资源，并将其全部纳入在单个事务中。

本地事务的一些常见示例在JMS和JDBC API中。
在JMS中，用户可以创建一个事务处理的会话，发送和接收消息，当消息的工作完成后，调用`Session.commit()`来告诉服务器它可以完成工作。
在数据库世界中，JDBC Connections默认自动提交查询。这对于一次性语句是很好的，但通常最好将一些相关的语句收集到一个批处理中，然后全部提交它们或者全部不提交。
在JDBC中，首先将Connection的`setAutoCommit()`方法设置为false，然后在批处理结束时显式调用`Connection.commit()`来执行此操作。
这两个API和其他许多API都提供了一个事务的`unit-of-work`的概念，它们可能会由客户端自行决定`committed, finalized, flushed`或以其他方式持久化。
API广泛不同，但概念是一样的。

全局事务完全不同。只要您希望多个资源参与到一个事务中，就应该使用它们。
可能会有这样的需求：也许你想发送一个JMS消息并写入数据库？ 或者，也许你想使用两种不同的JPA持久性上下文？ 
在全局事务设置中，第三方事务监控器在一个事务中纳入多个事务资源，准备提交
- 在此阶段，资源通常执行相当于`dry-run`提交；
- 然后最终提交每个资源；

这些步骤是大多数全局事务实现的基础，被称为两阶段提交（2PC）。如果一个提交失败（就像网络中断），则事务监视器会请求每个资源撤消或回滚最后一个事务。

全局事务比常规的本地事务显得更复杂，因为根据定义，它们必须涉及第三方代理，其唯一功能是仲裁若干事务资源之间的事务状态。
事务监视器还必须知道如何与每个事务资源进行通信。

因为这个代理还必须保持状态 
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
提供了可靠的基于AOP的支持。通过此支持，您不再需要`TransactionTemplate`
- 对于启用声明式事务管理注解的任何方法，事务管理会自动进行。

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

因此，如果您在使用Java EE应用程序服务器，但希望使用Spring的JTA支持，则可以使用以下命名空间配置支持正确（并自动）构造`JtaTransactionManager`：

```xml
<tx:jta-transaction-manager />
```

或者，您可以根据需要注册JtaTransactionManager bean实例，而不使用构造函数参数，如下所示：

```java
@Bean
public PlatformTransactionManager platformTransactionManager(){ 
    return new JtaTransactionManager();
}
```

无论哪种方式，JavaEE应用程序服务器的最终结果是，现在可以使用JTA以统一的方式管理事务，这要归功于Spring。

## Using Embeddable Transaction Managers

许多人选择在Java EE应用服务器之外使用JTA，原因很明显：Tomcat或Jetty更轻便，更快速，更便宜，更容易测试，业务逻辑通常不会存在于应用服务器中等。
现在在云服务中，这比以往更重要，其中轻量级，可扩展的资源已经成为标准，而且重量级的单片应用服务器根本无法扩展。

有很多开源和商业的独立的JTA事务管理器。在开源社区，您有像Java Open Transaction Manager（JOTM），JBoss TS，Bitronix Transaction Manager（BTM）和Atomikos等几种选择。

在这篇文章中，我们将介绍一种采用全局事务的简单方法。我们将专注于Atomikos特定的配置，但是还有一个例子演示了在源代码中使用Bitronix的相同配置。

在这个例子中，事务方法是一个简单的基于JPA的服务，它必须同时提交一个JMS消息。代码是典型的在线零售商购物车的假设结帐方法。代码如下所示：

```java
@Transactional
public void checkout(long purchaseId) {
    Purchase purchase = getPurchaseById(purchaseId);

    if (purchase.isFrozen()) 
      throw new RuntimeException(
         "you can't check out Purchase(#" + purchase.getId() + ") that's already been checked out!");

    Date purchasedDate = new Date();
    Set<LineItem> lis = purchase.getLineItems();
    for (LineItem lineItem : lis) {
        lineItem.setPurchasedDate(purchasedDate);
        entityManager.merge(lineItem);
    }
    purchase.setFrozen(true);

    this.entityManager.merge(purchase);
    log.debug("saved purchase updates");
    
    this.jmsTemplate.convertAndSend(this.ordersDestinationName, purchase);
    log.debug("sent partner notification");
}
```

该方法使用`@Transactional`注解来告诉Spring将其调用包含在事务中。该方法采用JPA和JMS。
所以这个工作跨越了两个事务资源，两者之间必须一致：数据库更新操作和JMS发送操作都成功，或者两者都回滚。

为了测试JTA配置是否可用，你可以简单地在最后一行抛出一个RuntimeException，如下所示：

```java
if (true) throw new RuntimeException("Monkey wrench!");
```

届时，数据库中的`purchase`实体应将其状态更改为冻结状态，并发送JMS消息。在最后一行抛出的异常将回滚这两个更改。
Spring的声明式事务管理将拦截异常，并使用配置的`JtaTransactionManager`自动回滚事务。然后，您可以验证这两个事件从未发生，并且它们不会反映在相应的资源中：
不会将JMS消息加入队列，并且JPA实体的数据库记录不会被更改。 我使用的测试用例是：

```java
package org.springsource.jta.etailer.store.services;

import org.apache.commons.logging.*;
import org.junit.*;
import org.junit.runner.RunWith;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import org.springframework.test.context.support.AnnotationConfigContextLoader;
import org.springsource.jta.etailer.store.config.*;
import org.springsource.jta.etailer.store.domain.*;
import javax.inject.Inject;
import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertTrue;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(loader = AnnotationConfigContextLoader.class,
       classes = {AtomikosJtaConfiguration.class, StoreConfiguration.class})
public class JpaDatabaseCustomerOrderServiceTest {

	private Log log = LogFactory.getLog(getClass().getName());

	@Inject private CustomerOrderService customerOrderService;
	@Inject private CustomerService customerService;
	@Inject private ProductService productService;

	@Test
	public void testAddingProductsToCart() throws Exception {
		Customer customer = customerService.createCustomer("A", "Customer");
		Purchase purchase = customerOrderService.createPurchase(customer.getId());
		Product product1 = productService.createProduct(
                 "Widget1", "a widget that slices (but not dices)", 12.0);
		Product product2 = productService.createProduct(
                 "Widget2", "a widget that dices (but not slices)", 7.5);
		LineItem one = customerOrderService.addProductToPurchase(
                 purchase.getId(), product1.getId());
		LineItem two = customerOrderService.addProductToPurchase(
                 purchase.getId(), product2.getId());
		purchase = customerOrderService.getPurchaseById(purchase.getId());
		assertTrue(purchase.getTotal() == (product1.getPrice() + product2.getPrice()));
		assertEquals(one.getPurchase().getId(), purchase.getId());
		assertEquals(two.getPurchase().getId(), purchase.getId());
		// this is the part that requires XA to work correctly
		customerOrderService.checkout(purchase.getId());
	}
}
```

测试是一个简单的事务脚本：购物者创建一个帐户，找到喜欢的东西，将它们作为订单项添加到购物车中，然后检出。
结帐方法将购物车的更改状态保存到数据库，然后发送触发JMS消息，通知其他系统有新订单。在这种结帐方法中，JTA是必不可少的。

## Configuring the Basic Services

我们有两个配置类
- JTA提供程序的特定代码，用于正确构建Spring的JtaTransactionManager实例，
- 其余的配置将保持静态，无论所选的PlatformTransactionManager策略如何。

我使用Spring的模块化Java配置从其他配置中分离出JTA提供商特定的配置类，因此您可以轻松地在特定于Atomikos的JTA配置和特定于Bitronix的JTA配置之间切换。

我们来看看StoreConfiguration类 - 无论你使用哪个事务管理器实现，它将是一样的。 

我只摘录了突出的部分，以便您可以看到哪些部分与JTA提供商特定的配置进行交互。

```java
package org.springsource.jta.etailer.store.config;

import org.hibernate.cfg.ImprovedNamingStrategy;
import org.hibernate.dialect.MySQL5Dialect;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.dao.annotation.PersistenceExceptionTranslationPostProcessor;
import org.springframework.jms.core.JmsTemplate;
import org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean;
import org.springframework.orm.jpa.vendor.Database;
import org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter;
import org.springframework.stereotype.Service;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.annotation.EnableTransactionManagement;
import org.springframework.transaction.jta.JtaTransactionManager;
import javax.inject.Inject;
import javax.transaction.TransactionManager;
import javax.transaction.UserTransaction;
import java.util.Properties;

@EnableTransactionManagement
@Configuration
@ComponentScan(value = "org.springsource.jta.etailer.store.services")
public class StoreConfiguration {

	// ... 	

	// this is a reference to a specific Java configuration class for JTA 
	@Inject private AtomikosJtaConfiguration jtaConfiguration ;

	@Bean
	public JmsTemplate jmsTemplate() throws Throwable{
		JmsTemplate jmsTemplate = new JmsTemplate(jtaConfiguration.connectionFactory());
		// ... 
	}

	@Bean
	public LocalContainerEntityManagerFactoryBean entityManager () throws Throwable  {
		LocalContainerEntityManagerFactoryBean entityManager = 
		   new LocalContainerEntityManagerFactoryBean();		
		entityManager.setDataSource(jtaConfiguration.dataSource());
		Properties properties = new Properties();
		// ... 
		jtaConfiguration.tailorProperties(properties);
		entityManager.setJpaProperties(properties);
		return entityManager;
	}

	@Bean
	public PlatformTransactionManager platformTransactionManager()  throws Throwable {
		return new JtaTransactionManager( 
                         jtaConfiguration.userTransaction(), jtaConfiguration.transactionManager());
	}
}
```

配置都是样板式代码 - 只是为任何JPA或JMS应用程序配置的普通对象。
配置类要执行其工作，需要访问`javax.jms.ConnectionFactory`，`javax.sql.DataSource`，`javax.transaction.UserTransaction`和`javax.transaction.TransactionManager`。
因为满足这些接口的对象的构造是特定于每个事务管理器实现的，所以这些bean定义在一个单独的Java配置类中，我们通过在StoreConfiguration类的顶部使用字段注入（@Inject private AtomikosJtaConfiguration jtaConfiguration）来引入。

我们的StoreConfiguration通过`@EnableTransactionManagement`注释开启自动事务处理。

我们使用Spring 3.1的`@PropertySource`注解（与环境抽象相关联）来访问services.properties中的键和值。 属性文件如下所示：

```properties
dataSource.url=jdbc:mysql://127.0.0.1/crm
dataSource.driverClassName=com.mysql.jdbc.Driver
dataSource.dialect=org.hibernate.dialect.MySQL5InnoDBDialect
dataSource.user=crm
dataSource.password=crm

jms.partnernotifications.destination=orders
jms.broker.url=tcp://localhost:61616
```

任何JTA配置最终要做的最重要的事是提供用于创建PlatformTransactionManager实例所使用的
特定于提供商的UserTransaction的引用，以及特定于提供商的TransactionManager的引用，如下所示：

```java
@Bean	
public PlatformTransactionManager platformTransactionManager() throws Throwable {
    UserTransaction userTransaction = jtaConfiguration.userTransaction() ;
    TransactionManager transactionManager = jtaConfiguration.transactionManager() ;
    return new JtaTransactionManager(  userTransaction, transactionManager );
}
```

## Configuring Atomikos

我们不会看到Bitronix和Atomikos实现的细节，

Atomikos配置的源代码在[这里](http://git.springsource.org/spring-samples/spring-samples/blobs/master/showcases/spring-and-jta/src/main/java/org/springsource/jta/etailer/store/config/AtomikosJtaConfiguration.java)，
Bitronix配置的源代码在[这里](http://git.springsource.org/spring-samples/spring-samples/blobs/master/showcases/spring-and-jta/src/main/java/org/springsource/jta/etailer/store/config/BitronixJtaConfiguration.java)。 

我们来剖析Atomikos的实现，so that we can breakdown the important players. 
Once you know what the players are，那么了解任何第三方JTA提供商的配置很简单。
Swapping out which configuration is used when you run the code is straightforward。

```java
package org.springsource.jta.etailer.store.config;

import com.atomikos.icatch.jta.UserTransactionImp;
import com.atomikos.icatch.jta.UserTransactionManager;
import com.atomikos.icatch.jta.hibernate3.TransactionManagerLookup;
import com.atomikos.jdbc.AtomikosDataSourceBean;
import com.atomikos.jms.AtomikosConnectionFactoryBean;
import com.mysql.jdbc.jdbc2.optional.MysqlXADataSource;
import org.apache.activemq.ActiveMQXAConnectionFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;
import org.springframework.core.env.Environment;

import javax.inject.Inject;
import javax.jms.ConnectionFactory;
import javax.sql.DataSource;
import javax.transaction.TransactionManager;
import javax.transaction.UserTransaction;
import java.util.Properties;

@Configuration
public class AtomikosJtaConfiguration {

	@Inject private Environment environment ;

	public void tailorProperties(Properties properties) {
		properties.setProperty( "hibernate.transaction.manager_lookup_class", 
				 TransactionManagerLookup.class.getName());
	}

	@Bean
	public UserTransaction userTransaction() throws Throwable {
		UserTransactionImp userTransactionImp = new UserTransactionImp();
		userTransactionImp.setTransactionTimeout(1000);
		return userTransactionImp;
	}

	@Bean(initMethod = "init", destroyMethod = "close")
	public TransactionManager transactionManager() throws Throwable {
		UserTransactionManager userTransactionManager = new UserTransactionManager();
		userTransactionManager.setForceShutdown(false);
		return userTransactionManager;
	}

	@Bean(initMethod = "init", destroyMethod = "close")
	public DataSource dataSource() {
		MysqlXADataSource mysqlXaDataSource = new MysqlXADataSource();
		mysqlXaDataSource.setUrl(this.environment.getProperty("dataSource.url"));
		mysqlXaDataSource.setPinGlobalTxToPhysicalConnection(true);
		mysqlXaDataSource.setPassword(this.environment.getProperty("dataSource.password"));
		mysqlXaDataSource.setUser(this.environment.getProperty ("dataSource.password"));

		AtomikosDataSourceBean xaDataSource = new AtomikosDataSourceBean();
		xaDataSource.setXaDataSource(mysqlXaDataSource);
		xaDataSource.setUniqueResourceName("xads");
		return xaDataSource;
	}

	@Bean(initMethod = "init", destroyMethod = "close")
	public ConnectionFactory connectionFactory() {
		ActiveMQXAConnectionFactory activeMQXAConnectionFactory = new ActiveMQXAConnectionFactory();
		activeMQXAConnectionFactory.setBrokerURL(this.environment.getProperty( "jms.broker.url")  );
		AtomikosConnectionFactoryBean atomikosConnectionFactoryBean = new AtomikosConnectionFactoryBean();
		atomikosConnectionFactoryBean.setUniqueResourceName("xamq");
		atomikosConnectionFactoryBean.setLocalTransactionMode(false);
		atomikosConnectionFactoryBean.setXaConnectionFactory(activeMQXAConnectionFactory);
		return atomikosConnectionFactoryBean;
	}
}
```

Atomikos提供了自己的`java.sql.DataSource`和`javax.jms.ConnectionFactory`包装器，可将任何本地`java.sql.DataSource`或`javax.jms.ConnectionFactory`修改为JTA（和XA）感知的包。

要告诉Hibernate如何参与Atomikos事务，我们必须设置一个属性--`hibernate.transaction.manager_lookup_class` - 在这种情况下，该类为`TransactionManagerLookup`。您将需要为任何JTA实现执行此操作。

最后，我们需要提供一个`javax.transaction.TransactionManager`实现和一个`javax.transaction.UserTransaction`实现。该类顶部的两个Bean是这两个接口的Atomikos实现，用于构建Spring的`JtaTransactionManager`实现，该实现是`PlatformTransactionManager`的一个实现。

`PlatformTransactionManager`实例又由我们的配置类上的`@EnableTransactionManagement`注释自动获取，并且用于在调用了`@Transactional`的任何方法时执行事务。

`javax.transaction.UserTransaction`实现和`javax.transaction.TransactionManager`实现的职责类似：`UserTransaction`是面向用户的API，`TransactionManager`是面向服务器的API。所有JTA实现都指定了`UserTransaction`实现，因为它是JavaEE的最低要求。`TransactionManager`不是必需的，并不总是在每个服务器或JTA实现中都可用。

对于熟悉JTA的人来说，使用`UserTransaction`，就像在JavaEE中以编程方式控制事务一样，有一些重大的差距，这可能是可以理解的，因为现在已经过时的假设是在近十年前J2EE首次设想的时候，没有人会想做事务管理没有EJB。

问题是一些操作，例如挂起事务（例如获取''requires new"语义），只能在`TransactionManager`上。该接口在JTA规范中是标准化的，但与`UserTransaction`不同，它不提供a well-known JNDI location或其他获取的方式。其他一些事情，例如控制隔离级别或服务器特定的"事务命名"（用于监视或其他目的）在JTA中根本不可能。

`TransactionManager`提供了诸如事务挂起和恢复等高级功能，因此大多数提供商也支持它。事实上，很多`javax.transaction.TransactionManager`实现可以在运行时转换为`javax.transaction.UserTransaction`实现。Spring知道这一点，并且对这点很聪明。If you define an instance of Spring’s JtaTransactionManager implementation with only a reference to a javax.transaction.TransactionManager, it will attempt to coerce a javax.transaction.UserTransaction instance out of it at runtime, as well. Atomikos, however, does not do this, and so we explicitly define a javax.transaction.UserTransaction instance and - to take better advantage of the more enhanced capabilities of a javax.transaction.TransactionManager - a separate javax.transaction.TransactionManager instance.

And, that’s it! You can have your cake, and eat it too with Spring. The Bitronix configuration looks similar as it is satisfying similar duties. You won’t have to tweak this code very often. It’s quite likely you can simply reuse the configuration presented here, adjusting the connection strings and drivers as appropriate.