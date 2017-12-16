## 关于WebMvc的配置方式
WebMvc的配置方式有以下三种：

1. 最暴力的配置方式就是**直接继承`WebMvcConfigurationSupport`**;

2. 比较温柔的方式就是**使用注解`@EnableWebMvc`**。
    
    这个注解是引入了`DelegatingWebMvcConfiguration`这样一个**代理配置类**，它**继承**了`WebMvcConfigurationSupport`。
    **用于检测和委派所有类型为`WebMvcConfigurer`的bean**，从而允许他们自定义由`WebMvcConfigurationSupport`提供的配置。
    
    所以，只要往容器里装入`WebMvcConfigurer`的自定义类就可以了。
    
3. Spring Boot配置方式

    Spring Boot自动配置类`WebMvcAutoConfiguration`里定义了`WebMvcAutoConfigurationAdapter`，也就是`WebMvcConfigurer`子类。
    
    ```java
    @Configuration
    @ConditionalOnWebApplication
    @ConditionalOnClass({ Servlet.class, DispatcherServlet.class,
            WebMvcConfigurerAdapter.class })
    @ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
    @AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
    @AutoConfigureAfter({ DispatcherServletAutoConfiguration.class,
            ValidationAutoConfiguration.class })
    public class WebMvcAutoConfiguration {
    
        ...
    
        // Defined as a nested config to ensure WebMvcConfigurerAdapter is not read when not
        // on the classpath
        @Configuration
        @Import(EnableWebMvcConfiguration.class)
        @EnableConfigurationProperties({ WebMvcProperties.class, ResourceProperties.class })
        public static class WebMvcAutoConfigurationAdapter extends WebMvcConfigurerAdapter {
           
        }
     }
    ``` 
    ```java
    public abstract class WebMvcConfigurerAdapter implements WebMvcConfigurer {
    
    }
    ```
    
    同样定义了`EnableWebMvcConfiguration`，是`DelegatingWebMvcConfiguration`子类，所以**连注解`@EnableWebMvc`也不需要了**。
    ```java
    /**
    	 * Configuration equivalent to {@code @EnableWebMvc}.
    	 */
    	@Configuration
    	public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration {
    ```
    
    注意：由于自动配置类`WebMvcAutoConfiguration`的引入条件是`@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)`

    所以，使用了上面方式1或者2时，是不会自动配置的，这样也就失去了自动配置类做的配置。
    

由此可见，尽量使用第3种方式，优先使用`application.properties`来配置，不够的地方，添加`WebMvcConfigurer`子类配置，只有实在无法满足要求时才使用2甚至1的方法。






 










