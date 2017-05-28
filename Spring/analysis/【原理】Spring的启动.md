1. 首先，对于**一个web应用**，其**部署在web容器中**，**web容器提供其一个全局的上下文环境**，这个上下文就是`ServletContext`，
    其**为后面的Spring IoC容器提供宿主环境**；
    
2. 其次，在`web.xml`中会提供有`ContextLoaderListener`。
    **在web容器启动时，会触发容器初始化事件，此时`ContextLoaderListener`会监听到这个事件**， 其`contextInitialized`方法会被调用，
    ```java
    public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
        public ContextLoaderListener() {
        }
    
        public ContextLoaderListener(WebApplicationContext context) {
            super(context);
        }
    
        public void contextInitialized(ServletContextEvent event) {
            this.initWebApplicationContext(event.getServletContext());
        }
    
        public void contextDestroyed(ServletContextEvent event) {
            this.closeWebApplicationContext(event.getServletContext());
            ContextCleanupListener.cleanupAttributes(event.getServletContext());
        }
    }
    ```
    在`contextInitialized`这个方法中，spring会初始化一个启动上下文，这个上下文被称为根上下文，即`WebApplicationContext`，这是一个接口类， 
    确切的说，其实际的实现类是`XmlWebApplicationContext`。这个就是**Spring IoC容器**，其对应的**Bean定义的配置由`web.xml`中的`context-param`标签指定**。 
    在这个IoC容器初始化完毕后，Spring以`WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`为属性Key，将其存储到ServletContext中，便于获取；
    ```java
    public class ContextLoader {
        private WebApplicationContext context;
        
        public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
            ...
            servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
            ...
        }
    }
    ```
    ```java
    public interface WebApplicationContext extends ApplicationContext {
        String ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE = WebApplicationContext.class.getName() + ".ROOT";
    ```
3. 再次，**`ContextLoaderListener`监听器初始化完毕后，开始初始化web.xml中配置的Servlet**，这个servlet可以配置多个，以**最常见的`DispatcherServlet`**为例， 
    这个servlet实际上是一个**标准的前端控制器**，**用以转发、匹配、处理每个servlet请求**。 
    **`DispatcherServlet`上下文在初始化的时候会建立自己的IoC上下文，用以持有spring mvc相关的bean**。 
    在建立`DispatcherServlet`自己的IoC上下文时，会利用`WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`先从`ServletContext`中获取之前的根上下文(即WebApplicationContext)作为自己上下文的parent上下文。 
    有了这个parent上下文之后，再初始化自己持有的上下文。

4. 这个**`DispatcherServlet`初始化**自己上下文的工作在其`initStrategies`方法中可以看到，**大概的工作就是初始化处理器映射、视图解析等**。
    ```java
    protected void initStrategies(ApplicationContext context) {
    		initMultipartResolver(context);
    		initLocaleResolver(context);
    		initThemeResolver(context);
    		initHandlerMappings(context);
    		initHandlerAdapters(context);
    		initHandlerExceptionResolvers(context);
    		initRequestToViewNameTranslator(context);
    		initViewResolvers(context);
    		initFlashMapManager(context);
    	}
    ```
    这个servlet自己持有的上下文默认实现类也是XmlWebApplicationContext。 
    初始化完毕后，spring以与servlet的名字相关(此处不是简单的以servlet名为Key，而是通过一些转换，具体可自行查看源码)的属性为属性Key，也将其存到ServletContext中，以便后续使用。
    ```java
    public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {
    
        public static final String SERVLET_CONTEXT_PREFIX = FrameworkServlet.class.getName() + ".CONTEXT.";
    
        protected WebApplicationContext initWebApplicationContext() {
            ...
            if (this.publishContext) {
                // Publish the context as a servlet context attribute.
                String attrName = getServletContextAttributeName();
                getServletContext().setAttribute(attrName, wac);
            }
            ...
        }
    
        /**
         * Return the ServletContext attribute name for this servlet's WebApplicationContext.
         * <p>The default implementation returns
         * {@code SERVLET_CONTEXT_PREFIX + servlet name}.
         * @see #SERVLET_CONTEXT_PREFIX
         * @see #getServletName
         */
        public String getServletContextAttributeName() {
            return SERVLET_CONTEXT_PREFIX + getServletName();
        }
    }
    ```
    
这样**每个servlet就持有自己的上下文，即拥有自己独立的bean空间**，同时**各个servlet共享相同的bean，即根上下文(第2步中初始化的上下文)定义的那些bean**。
