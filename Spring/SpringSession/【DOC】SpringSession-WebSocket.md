>原文：http://docs.spring.io/spring-session/docs/current/reference/html5/guides/websocket.html
>
>Version 1.3.1.RELEASE
>
>Last updated 2017-04-27 18:42:57 +00:00

本指南介绍如何使用Spring Session来确保WebSocket消息使您的HttpSession Keep-Alive。

>Spring Session的WebSocket支持仅适用于Spring的WebSocket支持。
>具体来说，它不能直接使用JSR-356。这是因为JSR-356没有用于拦截传入的WebSocket消息的机制。

## HttpSession Setup
第一步是将Spring Session与HttpSession进行整合。

这些步骤已在[HttpSession指南](http://docs.spring.io/spring-session/docs/current/reference/html5/guides/httpsession.html)中概述。

在继续之前，请确保您已经将Spring Session与HttpSession集成。

## Spring Configuration
在一个典型的Spring WebSocket应用程序中，用户将继承`AbstractWebSocketMessageBrokerConfigurer`。
      
例如，配置可能如下所示：

```java
@Configuration
@EnableScheduling
@EnableWebSocketMessageBroker
public class WebSocketConfig extends AbstractWebSocketMessageBrokerConfigurer {

        public void registerStompEndpoints(StompEndpointRegistry registry) {
                registry.addEndpoint("/messages").withSockJS();
        }

        @Override
        public void configureMessageBroker(MessageBrokerRegistry registry) {
                registry.enableSimpleBroker("/queue/", "/topic/");
                registry.setApplicationDestinationPrefixes("/app");
        }
}
```

我们可以轻松地更新我们的配置以使用Spring Session WebSocket支持。

例如：
>src/main/java/samples/config/WebSocketConfig.java

```java
@Configuration
@EnableScheduling
@EnableWebSocketMessageBroker
public class WebSocketConfig
                extends AbstractSessionWebSocketMessageBrokerConfigurer<ExpiringSession> { 

        protected void configureStompEndpoints(StompEndpointRegistry registry) { 
                registry.addEndpoint("/messages").withSockJS();
        }

        public void configureMessageBroker(MessageBrokerRegistry registry) {
                registry.enableSimpleBroker("/queue/", "/topic/");
                registry.setApplicationDestinationPrefixes("/app");
        }
}
```
Spring Session支持中，我们只需要改变两件事情：
- 我们继承`AbstractSessionWebSocketMessageBrokerConfigurer`，而不是继承`AbstractWebSocketMessageBrokerConfigurer`;
- 我们重命名`registerStompEndpoints`方法为`configureStompEndpoints`;

**`AbstractSessionWebSocketMessageBrokerConfigurer`在幕后做什么？**

- `WebSocketConnectHandlerDecoratorFactory`作为`WebSocketHandlerDecoratorFactory`添加到`WebSocketTransportRegistration`
    这样可以确保启动包含`WebSocketSession`的自定义`SessionConnectEvent`。
    当Spring Session终止时，`WebSocketSession`必须终止任何仍然打开的WebSocket连接。
- `SessionRepositoryMessageInterceptor`作为`HandshakeInterceptor`添加到每个`StompWebSocketEndpointRegistration`
    这样可以确保将会话添加到WebSocket属性中，使能够更新上次访问的时间。
- `SessionRepositoryMessageInterceptor`作为`ChannelInterceptor`添加到我们的`inbound ChannelRegistration`中
    这样可以确保每次接收到`inbound`消息时，我们的Spring Session最后访问的时间将被更新。
- `WebSocketRegistryListener`被创建为一个Spring Bean
    这样可以确保我们将所有Session id映射到相应的WebSocket连接。通过维护此映射，当Spring Session（HttpSession）终止时，我们可以关闭所有的WebSocket连接。

## websocket Sample Application
websocket示例应用程序演示如何使用Spring Session与WebSockets。

### Running the websocket Sample Application
您可以通过获取[源代码](https://github.com/spring-projects/spring-session/archive/1.3.1.RELEASE.zip)并调用以下命令来运行示例：

为了测试会话到期，您可能希望通过在启动应用程序之前从以下文件中删除注释来将会话期限更改为1分钟（默认为30分钟）：

>src/main/java/samples/config/WebSecurityConfig.java

```java
@Configuration
@EnableWebSecurity
@EnableRedisHttpSession // (maxInactiveIntervalInSeconds = 60)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
```

>要使示例工作，必须在本地主机上安装Redis 2.8+，并使用默认端口（6379）运行。 或者，您可以更新`LettuceConnectionFactory`以指向Redis服务器。

```text
$ ./gradlew :samples:websocket:bootRun
```

您现在应该可以访问http://localhost:8080/

## Exploring the websocket Sample Application
尝试使用该应用程序。使用以下信息认证：
- Username rob
- Password password

现在点击**Login**按钮。您现在应该以用户**rob**身份认证。

打开匿名窗口并访问http://localhost:8080/

系统将提示您登录表单。使用以下信息认证：

- Username luke
- Password password

现在发送一条从rob到luke的消息。该消息应该会发送。

等待两分钟，然后再次尝试发送一条从rob到luke的消息。您将看到该消息不再发送。

>为什么两分钟？
>
>Spring Session将在60秒后到期，但Redis的通知不能保证在60秒内发生。 
>为了确保Socket套接字在合理的时间内关闭，Spring会话在每分钟00秒运行一个后台任务，强制清除任何过期的会话。 
>这意味着您需要在WebSocket连接终止之前最多等待两分钟。

尝试访问http://localhost:8080/将会提示您再次进行身份认证。这表明会话正确到期。

现在重复相同的练习，每30秒钟从**每个用户**发送一条消息，而不是等待两分钟。您将看到消息继续发送。 

尝试访问http://localhost:8080/不会再提示您进行身份验证。这表明会话Keep-Alive。

>只有从用户发送的消息保持会话活动。这是因为只有来自用户的消息才意味着用户活动。收到的消息并不意味着活动，因此不会更新会话到期。
