本指南介绍如何使用Spring Session，基于Java配置透明地利用Redis来支持Web应用程序的HttpSession。

## Updating Dependencies
在使用Spring Session之前，必须确保更新依赖项。 如果您使用Maven，请确保添加以下依赖项：

>pom.xml

```xml
<dependencies>
    <!-- Aggregator for Spring Session and Spring Data Redis -->
    <!-- 
        注意：该依赖是一个Pom，它会自动导入下面的依赖：
            commons-pool2、spring-data-redis、spring-session、jedis
    -->
    <dependency>
            <groupId>org.springframework.session</groupId>
            <artifactId>spring-session-data-redis</artifactId>
            <version>1.3.1.RELEASE</version>
            <type>pom</type>
    </dependency>
    <dependency>
            <groupId>biz.paluch.redis</groupId>
            <artifactId>lettuce</artifactId>
            <version>3.5.0.Final</version>
    </dependency>
    <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-web</artifactId>
            <version>4.3.4.RELEASE</version>
    </dependency>
</dependencies>
```

## Spring Java Configuration
添加所需的依赖后，我们可以创建我们的Spring配置。

Spring配置负责创建一个使用Spring Session支持的实现替换HttpSession实现的Servlet Filter。添加以下Spring配置：

```java
@EnableRedisHttpSession 
public class Config {

    @Bean
    public LettuceConnectionFactory connectionFactory() {
            return new LettuceConnectionFactory(); 
    }
}
```
- `@EnableRedisHttpSession`注释创建一个实现Filter接口的名为springSessionRepositoryFilter的Spring Bean。
    过滤器负责使用Spring Session支持的实现替换HttpSession实现。在这种情况下，Spring Session由Redis支持。
- 我们创建一个将Spring Session连接到Redis Server的`RedisConnectionFactory `。
    我们配置连接本地localhost，默认端口（6379）。
    有关配置Spring Data Redis的更多信息，请参阅[参考文档](http://docs.spring.io/spring-data/data-redis/docs/current/reference/html/)。

## Java Servlet Container Initialization
我们的Spring配置创建了一个实现了`Filter`接口，名为`springSessionRepositoryFilter`的Spring Bean。
`springSessionRepositoryFilter` bean负责使用`Spring Session`支持的自定义实现替换`HttpSession`。

为了使我们的过滤器能够做到这一点，Spring需要加载我们的Config配置类。

最后，我们需要确保我们的Servlet容器（即Tomcat）使用我们的`springSessionRepositoryFilter`拦截每个请求。

幸运的是，Spring Session提供了一个名为`AbstractHttpSessionApplicationInitializer`的实用程序类，这两个步骤非常简单。 

>src/main/java/sample/Initializer.java

```java
public class Initializer extends AbstractHttpSessionApplicationInitializer { 

    public Initializer() {
            super(Config.class); 
    }
}
```
>我们的类的名称 (Initializer) 并不重要。 重要的是我们继承AbstractHttpSessionApplicationInitializer。

第一步是继承`AbstractHttpSessionApplicationInitializer`。 
这样可以确保名称为`springSessionRepositoryFilter`的Spring Bean在Servlet容器中为每个请求注册。

`AbstractHttpSessionApplicationInitializer`还提供了一种机制，可轻松确保Spring加载我们的Config。

## httpsession Sample Application
### Running the httpsession Sample Application
您可以通过获取[源代码](https://github.com/spring-projects/spring-session/archive/1.3.1.RELEASE.zip)并调用以下命令来运行示例：

>要使示例工作，必须在本地主机上安装Redis 2.8+，并使用默认端口（6379）运行。 或者，您可以更新`LettuceConnectionFactory`以指向Redis服务器。

```text
$ ./gradlew :samples:httpsession:tomcatRun
```

您现在应该可以访问http://localhost:8080/

### Exploring the httpsession Sample Application
尝试使用该应用程序。 使用以下信息填写表单：

- Attribute Name: username
- Attribute Value: rob

现在单击`Set Attribute`按钮。 您现在应该看到表中显示的值。

### How does it work?
我们与`SessionServlet`中标准的`HttpSession`进行交互，如下所示：

>src/main/java/sample/SessionServlet.java

```java
public class SessionServlet extends HttpServlet {

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
                    throws ServletException, IOException {
            String attributeName = req.getParameter("attributeName");
            String attributeValue = req.getParameter("attributeValue");
            req.getSession().setAttribute(attributeName, attributeValue);
            resp.sendRedirect(req.getContextPath() + "/");
    }

    private static final long serialVersionUID = 2878267318695777395L;
}
```
我们实际上是在Redis中保留这些值，而不是使用Tomcat的`HttpSession`。 Spring会话在您的浏览器中创建一个名为`SESSION`的cookie，其中包含会话的ID。继续查看Cookie。

如果您愿意，可以使用redis-cli轻松删除会话。例如，在基于Linux的系统上，您可以键入：
```text
$ redis-cli keys '*' | xargs redis-cli del
```

或者，您也可以删除指定的键。 在您的终端中输入以下内容，确保用您的SESSION cookie的值替换7e8383a4-082c-4ffe-a4bc-c40fd3363c5e：

```text
$ redis-cli del spring:session:sessions:7e8383a4-082c-4ffe-a4bc-c40fd3363c5e
```

现在通过http://localhost:8080/访问应用程序，并观察我们添加的属性将不再显示。